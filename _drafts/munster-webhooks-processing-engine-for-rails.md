---
layout: post
title: Munster - Webhooks processing engine for Rails
---
By the time of writing this article, I had already written webhook processing logic at least 10 times for different companies and clients. I and [Julik](https://blog.julik.nl/) had implemented one recently at our [current place of employment](https://cheddar.me). And guess what? We had to do it one more time for a microservice we were about to erect next to our majestic monolith.

Our combined experience in the digestion of webhooks, had already produced a reasonable generic solution. Should we just copy some files over to a microservice and duplicate that code? Nah, let's save the world from wasting those countless hours of re-implementing webhooks and processing them over and over again. 

Let's improve it for everyone and extract existing code into a gem.

## Enter The "Munster"
Munster is a webhooks processing engine for Rails. And you heard that right; it's only processing and not emitting. 

Services that send webhooks would love it if you accepted those as fast as possible and responded with a 200 status code. And we do that really fast by not processing webhooks right away; Munster just stores them first. As a second step, we enqueue a job in the async processor to do all the heavy lifting in a background process. This helps to:

- Equally manage load on servers. 
- Background process is not subject to any request timeouts. 
- If processing fails, we will have a webhook safely stored in our database for later re-processing.

It's not always possible to replay webhooks from a third party, not even if you ask politely. So storing them first, instead of processing them right away, seems like a great architectural decision. 

That's the gist of it; everything else depends on how you implement it.

## Getting started
As with any other gem, you'd first want to add `gem 'munster'` into a `Gemfile` and run `bundle install`. This would add a generator task to a Rails project, so run `rails g munster:install`.

It will create the required migration and an initializer file at app/initializers/munster.rb. This initializer would expect you to define at least one `active_handlers` hash for it. I will quickly run you through the process and give you an example.

### Mounting

Now let's map Munster in the `config/routes.rb` file.

You can mount a webhook engine on a subdomain (like `webhooks.yourdomain.com/:service_id`).
```
scope as: "webhooks", constraints: "webhooks.yourdomain.com" do
    mount Munster::Engine, at: "/", as: ""
end
```

Or you can mount the webhook's engine to the main domain (`yourdomain.com/webhooks/:service_id`):

`mount Munster::Engine => "/webhooks"`

### Defining a Handler

Next step, is to define a webhook handler. For the sake of an example, let's create a Customer.io handler at `app/webhooks/customer_io_handler.rb`, with a following content:

```
class Webhooks::CustomerIoHandler < Munster::BaseHandler
    def process(webhook)
      return if webhook.status.eql?("processing")
      webhook.processing!
    
      json = JSON.parse(webhook.body, symbolize_names: true)
    
      case json[:metric]
      when "subscribed"
        ActiveRecord::Base.transaction do
          user = User.find(json.dig(:data, :customer_id))
          user.update!(subscribed: true)
          
          webhook.processed!
        end
      when "unsubscribed"
        ActiveRecord::Base.transaction do
          user = User.find(json.dig(:data, customer_id))
          user.update!(subscribed: false)
          
          webhook.processed!
        end
      else
        webhook.skipped!
      end
    rescue => error
      webhook.error!
    
      raise error
    end
    
    def extract_event_id_from_request(action_dispatch_request)
      JSON.parse(action_dispatch_request.body.read).fetch("event_id")
    end
    
    # Verify that request is actually comming from customer.io
    # @see https://customer.io/docs/api/webhooks/#section/Securely-Verifying-Requests
    def valid?(action_dispatch_request)
      xcio_signature = action_dispatch_request.headers["HTTP_X_CIO_SIGNATURE"]
      xcio_timestamp = action_dispatch_request.headers["HTTP_X_CIO_TIMESTAMP"]
      request_body = action_dispatch_request.body.read
      string_to_sign = "v0:#{xcio_timestamp}:#{request_body}"
    
      hmac = OpenSSL::HMAC.hexdigest("SHA256", 'customer_io_webhook_signing_key', string_to_sign)
    
      Rack::Utils.secure_compare(hmac, xcio_signature)
    end
end
```

This pseudocode stores a local value per user called "subscribed". If someone unsubscribed from customer.io, this fact would be stored in our database. The same would happen if the user subscribed. We're rewriting three methods from [BaseHandler](https://github.com/cheddar-me/munster/blob/main/lib/munster/base_handler.rb):

- `valid?` method verifies that the webhook indeed comes from customer.io
- `process` method defines how we want to process data
- `extract_event_id_from_request` method defines how to extract an ID from a webhook; by default, this method generates a random UUID.

There are more methods that could be redefined and tweaked, but these three are most commonly used.

And lastly, to start using this handler, we need to define it in the initialization file. All keys from `active_handlers` will be mapped to :service_id in URLs.

```
require_relative '../../app/webhooks/customer_id_handler.rb'

Munster.configure do |config|
    config.active_handlers = {
        customer_id: Webhooks::CustomerIoHandler
    }
end
```

## Final note
We went through the basic webhook endpoint setup with Munster, but it offers a bit more than that. This little framework is especially beneficial in cases where you have more than one webhook endpoint. Once you set it up in your project , it's just a matter of defining a single handler to accept new types of webhooks.

The core of this is already battle-tested at cheddar.me, but we're still working on fleshing out the details and would love your feedback.

