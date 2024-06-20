---
layout: post
title: Munster - Webhooks processing engine for Rails
date: 2024-06-18 17:52 +0200
---
By the time of writing this article, I had already written webhook processing logic at least 10 times for different companies and clients. Together with [Julik](https://blog.julik.nl/) we have implemented one recently at our [current place of employment](https://cheddar.me). And guess what? Once a second service had to be built, it needed to accept webhooks too.

Our combined experience in the ingestion of webhooks had already produced a reasonable generic solution. Should we just copy some files over to a microservice and duplicate that code? Nah, let's save the world from wasting those countless hours of re-implementing webhooks over and over again!  This darn well could be a gem!

## Enter "Munster"

Munster is a webhooks processing engine for Rails. And you heard that right: it's only accepting and processing, not sending.

Services that send webhooks would love it if you accepted those as fast as possible and responded with a 200 status code. Pretty much always, actually - because if your service refuses to accept the webhooks they send you they will likely stop retrying. Important business events could then get lost, you would need to examine the retry policies of every webhook sender, etc.

And if you didn't manage to receive a webhook correctly, it is usually a hassle to ask the sending service to re-deliver the missed webhooks to you in bulk. Some (like Stripe) provide facilities to replay their event feed in a "pull" manner, but most do not.

How to make that happen well? The answer is pretty simple: do not process the webhook inline. Verify it is coming from the sender (most good webhooks use some form of signature that you can verify using a shared secret or a public key), and spool the webhook for processing using your background jobs. Processing webhooks asynchronously has many other advantages:

- Equally manage load on servers.
- Background process is not subject to any request timeouts.
- If processing fails, we will have a webhook safely stored in our database for later re-processing or analysis.

Munster is an engine which will provide you the following facilities:

* A small abstraction for building webhook receiving _handlers._ A handler is a small object with just a handful of methods that you need to implement - normally those would be `valid?` and `process`
* A Rails engine that can be mounted into your application and handles webhooks from multiple senders together - every webhook sender would get its own _handler_ definition. So you would have a handler for Stripe, a handler for Revolut, a handler for Github - and any other services you might want to receive webhooks from
* A background job class which calls `process` asynchronously, from your job queue.

## Getting started

As with any other gem, you'd first want to add `gem 'munster'` into a `Gemfile` and run `bundle install`. This would add a generator task to a Rails project, so run `rails g munster:install`. Then run `rails db:migrate` so that the table used for received webhooks gets created.

It will create the required migration and an initializer file at `app/initializers/munster.rb`. This initializer would expect you to define at least one `active_handlers` hash for it. I will quickly run you through the process and give you an example.

### Mounting

Munster is an engine and a Rack app,  so it can be mounted in your Rails routes file. Inside `config/routes.rb` you can mount a webhook engine on a subdomain (like `webhooks.yourdomain.com/:service_id`):

```ruby
scope as: "webhooks", constraints: "webhooks.yourdomain.com" do
    mount Munster::Engine, at: "/", as: ""
end
```
Or on the main domain (`yourdomain.com/webhooks/:service_id`):

`mount Munster::Engine => "/webhooks"`

Having a separate subdomain for receiving webhooks can be useful if you have very specific security requirements, or if you would like to have a separate load balancer fronting your webhook receiving endpoint.

### Defining a Handler

The next step is to define a webhook handler. For the sake of an example, let's create a handler for Customer.io at `app/webhooks/customer_io_handler.rb`. The handler will take care of handling two specific metrics from Customer.io - `subscribed` and `unsubscribed`. We want to store a local value per user called "subscribed" – once a user unsubscribes using customer.io, we want to record this information in our app database. Same for when a user subscribes.

```ruby
class Webhooks::CustomerIoHandler < Munster::BaseHandler
    def process(webhook)
      return if webhook.status.eql?("processing")
      webhook.processing!

      # The webhook body gets stored as bytes, so senders may deliver
      # you binary data on the endpoint - it does not have to be JSON
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

We're rewriting three methods from [BaseHandler](https://github.com/cheddar-me/munster/blob/main/lib/munster/base_handler.rb):

- The `valid?` method verifies that the webhook indeed comes from customer.io. This method runs inline, before we persist the webhook.
- The `process` method defines how we want to process data
- The `extract_event_id_from_request` method defines how to extract an ID from a webhook, and by default it will generate a random UUID.

The `extract_event_id_from_request` is a very important feature. A lot of webhook senders send you unique webhooks, but those webhooks may arrive out of order, or arrive twice. Networks being unreliable, the retries being too aggressive at the sender side, you name it. Munster will use this event ID to deduplicate your webhook - if you receive the same data more than once, just one webhook will be persisted and processed. This will also protect your infrastructure if, for some reason, the webhook sender happens to DoS you with repeated deliveries.

There are more methods that could be redefined and tweaked, but these three are most commonly used.

And lastly we need to mount our handler. We use the "service ID", which will also be the URL path component for your webhook. For our handler, we will use "customer-io" (our URL to paste into the Customer.io configuration will thus be `https://your-app.example.com/webhooks/customer-io`):

```
require_relative '../../app/webhooks/customer_id_handler.rb'

Munster.configure do |config|
    config.active_handlers = {
        "customer-io" => Webhooks::CustomerIoHandler
    }
end
```

## Final note

We went through the basic webhook endpoint setup with Munster, but it offers a bit more than that. This little framework is especially beneficial in cases where you have more than one webhook endpoint. Once you set it up in your project , it's just a matter of defining a single handler to accept new types of webhooks.

The core of this is already battle-tested at [cheddar.me](https://www.cheddar.me), but we're still working on fleshing out the details and would love your feedback! You can find Munster at [https://github.com/cheddar-me/munster](https://github.com/cheddar-me/munster)
