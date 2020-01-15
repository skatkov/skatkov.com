---
title: Fully automated automated testing (doom guy edition)
published: true
description: 
layout: page
tags: ruby, minitest, tdd
---

![Woohoo, Doom guy in notifications!](https://thepracticaldev.s3.amazonaws.com/i/879q596j4z6igz2j52fc.png)


I hate repetitive work with passion and I try to avoid it as much as possible. While browsing twitter I randomly stumbled on [Sam Saffron video, there he talks about his love of improving own workflow and building dev tools](https://www.youtube.com/watch?v=aP5NNkzb4og&feature=youtu.be&t=2572).

One off first point Sam mentions there is **autospec** and demonstrates how it works. I had really mixed feelings about this particular part. I felt relieved that my testsuite takes less than 5 minutes, but at the same time, I had this nagging feeling inside me too...

My workflow consists of following stages:

1. do small code changes
2. run tests
3. rollback or fix test if something failed
4. continue to step 1

It's hard to believe, but most developers do this all day. Some people manually execute tests, but I just run automated tests manually.  In my case it takes 20 second for ~500 assertions to happen... It feels repetitive really fast. 

So... And here is Sam... He doesn't execute testsuite routinely through the day? Doesn't pick specific files for testing? Yes, please, give me the same. But I felt even worse then I understood that [autospec](https://github.com/arnvald/autospec) doesn’t work with [minitest](https://github.com/seattlerb/minitest). I had no desire to change to rspec.

So what do other minitest users do in this case? What would you do? If your using [guard](https://github.com/guard/guard) already, you may want to pick [guard-minitest](https://github.com/guard/guard-minitest). But it just runs all tests, without focusing on broken one. If it is okey with you - then go on and use it. if you want something better you can read further.

## Let's use minitest-autotest

It’s pretty easy to add. Just open your **Gemfile** and add these lines:
```ruby
    group :test do
      gem "minitest-autotest”
    end
```

Create a simplest possible configuration file in root of your project and call it **.autotest**
```ruby
    require 'autotest/restart'

    Autotest.add_hook :initialize do |at|
      at.testlib = "minitest/autorun"
    end
```

You don’t really need it, it “just  works” (c) without it. But we'll need it later, there is obviously something missing here. I don’t want to be checking my console all the time, to be sure that I didn’t broke anything.

## Better notification to the rescue!

We already talked about guard. Guard wiki contains very [nice summary about system notifications and libraries](https://github.com/guard/guard/wiki/System-notifications) that we can use. Since I use OSX - my two options are growl and [terminal-notifier](https://github.com/julienXX/terminal-notifier). I don't really feel like paying for growl, so terminal notifier seems like a perfect candidate for a job. Let’s add more code, shall we?

Our **Gemfile** should have one additional line:
```ruby
    group :test do
      gem "minitest-autotest"
      gem "terminal-notifier"
    end
```

And our **.autotest** code becomes a bit more interesting:
```ruby
    require 'autotest/restart'
    require 'terminal-notifier'

    Autotest.add_hook :initialize do |at|
      at.testlib = "minitest/autorun"
    end

    Autotest.add_hook :ran_command do |at|
      if at.failures.count > 0
        TerminalNotifier.notify(failed_message(at.failures.count),
          :title => 'minitest-autotest'
         )
      end
    end

    def failed_message(count)
      count.eql?(1) ?  "#{count} test failed" :  "#{count} tests failed"
    end
```

This is already much better - now we receive notification every time tests fail. But this sounds like a problem:

- we're bombarded by notifications. autotest focuses on erroneous test and runs it indefinitely (generating notifications every single time)
- If we rollback our code, we have to again verify in console that spec's passed. Would be nice, if we don't have to do this.

## Let's make notifications more intelligent (without machine learning...)
```ruby
    require 'autotest/restart'
    require 'terminal-notifier'

    SKIP_MESSAGES = 5

    Autotest.add_hook :initialize do |at|
      at.testlib = "minitest/autorun"
      @counter = 0
    end

    Autotest.add_hook :all_good do |at|
      if @counter > 0
        TerminalNotifier.notify("All fixed!",
          :title => 'minitest-autotest'
        )

        @counter = 0
      end
    end

    Autotest.add_hook :ran_command do |at|
      if at.failures.count > 0
        if @counter.eql?(0) || @counter.eql?(SKIP_MESSAGES+1)
          TerminalNotifier.notify(failed_message(at.failures.count),
            :title => 'minitest-autotest'
          )

          @counter = 1
        else
          @counter += 1
        end
      end
    end

    def failed_message(count)
      count.eql?(1) ? "#{count} test failed" : "#{count} tests failed"
    end
```
So now we added a simple counter. Based on this counter we can determine:

* If we already shown error notifications and present 'Wohoo, everything is fixed' notification
* We can skip some of notifications, not to be overly annoying

![Not a cool version of notifications](https://thepracticaldev.s3.amazonaws.com/i/dh1dhf89vdj1qp2uwxsp.png)

It seems like this is almost perfect for my day-to-day activities. But lacks coolness..


## Doom guy enters the stage..

If your my age, you will never forget about epic game called Doom. This could also be a purely Estonian problem, because Jarmo Pertman (who is my age and Estonian as I am) long time ago wrote a [doom-guy-bleeding indicator plugin for old version of autotest](https://github.com/jarmo/autotest-doom) that doesn't work nowadays.

Let's use his work for good - [steal some of images he had there](https://github.com/jarmo/autotest-doom/tree/master/images) and place them into `/vendor/doomguy/` folder. And steal some of his code (thank god it's open source). Let's sum all of this:
```ruby
    require 'autotest/restart'
    require 'terminal-notifier'

    SKIP_MESSAGES = 5
    IMAGE_PATH = File.dirname(__FILE__) + "/vendor/doomguy/"

    Autotest.add_hook :initialize do |at|
      at.testlib = "minitest/autorun"
      @counter = 0
    end

    Autotest.add_hook :all_good do |at|
      if @counter > 0
        TerminalNotifier.notify("All fixed!",
          :title => 'minitest-autotest',
          :appIcon => IMAGE_PATH + "pass.png"
        )

        @counter = 0
      end
    end

    Autotest.add_hook :ran_command do |at|
      if at.failures.count > 0
        if @counter.eql?(0) || @counter.eql?(SKIP_MESSAGES+1)
          count = [(9 + at.failures.count) / 10 * 10, 50].min

          TerminalNotifier.notify(failed_message(at.failures.count),
            :title => 'minitest-autotest',
            :appIcon => IMAGE_PATH + "fail#{count}.png"
          )

          @counter = 1
        else
          @counter += 1
        end
      end
    end

    def failed_message(count)
      count.eql?(1) ? "#{count} test failed" : "#{count} tests failed"
    end
```

So, now I'm fully happy with result and productive!

![Woohoo, Doom guy in notifications!](https://thepracticaldev.s3.amazonaws.com/i/879q596j4z6igz2j52fc.png)

Happy testing!