---
title: Running a newsletter with better metrics
layout: post
description: I found myself running a semi-successful curated newsletter. What metric should I base this business on? Thinking out-loud.
---

![Got more metrics?](https://thepracticaldev.s3.amazonaws.com/i/fbght261a12ooqebtekq.jpeg)

# Too-many-metrics paranoia
In one of Gojko Adzic's books on project management, he argued about the importance of metrics. But as an engineer at heart, he vouched for **right metric**, not just any metric...

There was one thought that resonated with me deeply -- that a lot of people fall into the fallacy of tracking wrong metrics, because it's simple to track. _Too-many-metrics paranoia_ has been following me ever since I've read the book. And I mostly applied it to project and team management. 

As an example, I completely given up measuring individual metrics with teams, only team-wide metric makes sense. Even if you're evaluating team member performance, it's always wrong to use individual metrics taken outside of team context.

As example: One person could be cracking out more issues than anyone, but by doing so.. creates a bunch of work for QA engineers to test and for fellow developers to fix.

# Running a newsletter
I'm currently running a [curated newsletter for amazon sellers](https://www.fbamonthly.com). Project is less than a year old. We've been looking for a similar service - found non. So we started it.

In our best month we did $250. But with [SaaS project that we run on a side](https://www.ashop.co) to get same amount of money it took us much more effort. Can anyone guess, what we did to obtain our first paying customers for this newsletter? Nothing, they just wrote us. 

I never considered newsletter as a possible business model, but I've been proven wrong on so many levels about that. I'm really new in the field, my first year... What do I know?

Based on my short experience, I hear these questions all the time:
- How much **subscribers** do you have? 
- What's your email **open rate**?

These questions are just so wide-spread, that it feels like an industry standard. However, after careful consideration, they don't make much sense to me personally.


# Shit-talking on industry standard
If you had to choose only one metric, would you really pick one of these two? Would it answer all the questions? I came to personal conclusion, that both of those seem inacurate.

If newsletter would extend media platform beyond plain emails, **subscriber count** would make much less sense. If I had a websites with all my newsletters, mixing it up with unique visitors or returning visitors would make it even more confusing.

In essence, newsletter is a matter of content amplification, so we do exactly that by distributing on social networks. Most of those "views" never count towards **open rate**. And my bet is, that a lot of those social group members do follow us, but do not wish to subscribe.

And another issue...

One of the issues sent as a part of a newsletter in a email service of my choice -- ProtonMail:

![Alt Text](https://thepracticaldev.s3.amazonaws.com/i/7vh41g0ug022rzyt4nur.png)

There's an image missing, just because it goes bundled with tracking scripts -- proton blocks it. It also doesn't register as 'Email Open' event.

It's not only ProtonMail nowadays - Firefox blocks trackers now by default, Brave did that for a while. 

So... metrics that I provide to my prospects are potentially skewed or just plain wrong.


# Theory

My answer to a dilemma I shared, is still untested. And there is no data to back this up. So I'll call it a theory and please do feel free to criticize.

Advertiser tries to estimate engagement levels with our reader base. All this estimation happens mostly based on amount of eye balls that hide behind each subscriber.

But engagement metrics really matter here and main tracking point here is link clicks. To avoid all the blocking -- basic url shortener with support for analytics will close the deal.

Anyone here runs newsletters? What kind of metrics have you went with, if not for open rate and subscriber count? Would love some feedback!