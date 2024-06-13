---
layout: post
title: Unified Error Reporter Interface in Rails
---
This is a post about a Unified Error Reporter Interface that was added to Rails from version 7 - an "ActiveJob" or  "Active Cache" of error reporting. A couple of times in my experience, there was a need for such an interface. But when such need appeared at first - such an interface didn't exist. The second time, I didn't know it existed.

In hindsight, the idea seems obvious. And I, personally, implemented something similar at least once. It was difficult to imagine that any framework would have a "unified error reporting interface", but we're talking about Rails here, a "batteries included framework"!
## Interface
Unified error reporting interface could be found at `Rails.error`.

`.handle` method would swallow an error
```ruby
Rails.error.handle { raise "test" }
```

`.record` will re-raise an error
```ruby
Rails.error.record { raise "test" }
```

`.report` is a lower level method that both of those method use underneath
```ruby
begin
  raise "This is an error"
rescue => error
  Rails.error.report(error, handled: true, severity: :warning)
  false
end
```

There is also a convenient way to set a request-wide context, that will be passed to all errors.
```ruby
Rails.error.set_context('namespace': "internal")
```

## Support with Error Monitoring services
My small research has shown that support for this unified interface has not been widely adopted so far.  Even services that implement support for unified error reporter, don't support all of its methods.

Often, `Rails.error.record` gets suppressed. Since it's re-raising an error - monitoring services would rather avoid duplicated errors and use error reported by their APM client, then one reported by rails (since it doesn't have as much context).

This is my rough estimation how services support this feature

| Service     | Rails.error.report | Rails.error.record | Rails.error.handle | Notes                                                                                                              |
| ----------- | ------------------ | ------------------ | ------------------ | ------------------------------------------------------------------------------------------------------------------ |
| Honeybadger | ✔️                 | ❌                  | ✔️                 | [Docs](https://docs.honeybadger.io/lib/ruby/integration-guides/rails-exception-tracking/#the-rails-error-reporter) |
| Sentry      | ✔️                 | ✔️                 | ✔️                 | [CHANGELOG](https://github.com/getsentry/sentry-ruby/blob/master/CHANGELOG.md)                                     |
| AppSignal   | ✔️                 | ❌                  | ✔️                 | [Rails Docs](https://docs.appsignal.com/ruby/integrations/rails.html)                                              |
| Bugsnag     | ❌                  | ❌                  | ❌                  |                                                                                                                    |
| Rollbar     | ❌                  | ❌                  | ❌                  |                                                                                                                    |
| Raygun      | ✔️                 | ✔️                 | ✔️                 | [PR](https://github.com/MindscapeHQ/raygun4ruby/pull/178) (since 4)                                                |
| NewRelic    | ❌                  | ❌                  | ❌                 |                                                                                                                    |
| Scout       | ❌                  | ❌                  |❌                 |             |

## Going beyond intented use
Original pitch in rails issues was focused on improving the lives of maintainers of Error Monitoring services and rails core developers. It was hard to hookup into all layers of rails to gather all possible errors and warnings that Rails was raising. It was equally hard for rails-core team to communicate errors, that didn't end up buried somewhere in logs where nobody sees them.

We see that Monitoring Services are not in a huge hurry to implement support for this reporter. But rails developers are already actively integrating it to catch all the difficult to spot issues. But I would like to highlight that there are two additional actors who can benefit here.

- Gem writers (mainly those that include Railtie) - same as Rails maintainer, gem developers do have a need to surface internal errors. Why not use a similar interface for that?
- Application developers - some teams didn't settle yet on Error Monitoring service and using unified error reporting interface could work during this period of uncertainty

While coverage by services is not great, you might want to go with a separate gem named [safely](https://github.com/ankane/safely) that offer a similar unified interface. This gem already works with all those service mentioned here and even more. But it's still an additional dependency and that's not always desirable.

## Additional resources
- Rails Guides: [https://guides.rubyonrails.org/error_reporting.html](https://guides.rubyonrails.org/error_reporting.html)
- Source: [error_reporter.rb](https://github.com/rails/rails/blob/main/activesupport/lib/active_support/error_reporter.rb)
- PR: [https://github.com/rails/rails/pull/43625](https://github.com/rails/rails/pull/43625)
- Github Issue: [https://github.com/rails/rails/issues/43472](https://github.com/rails/rails/issues/43472)
