---
layout: post
title: Lessons from building another URL shortener
date: 2020-03-26 18:17 +0400
subtitle: After careful consideration, I’ve highlighted the 3 most important things to consider in a project like this.
---


I'm a software consultant working mostly with the web.

I secretly wanted to build an URL shortener for quite some time.  It's one of those projects that seems relatively easy at first, but during my work as a consultant, I stumbled upon in-house solutions that caused more harm in the long term, than it was worth.

I’ve always wondered what my solution would be to the problems I came across and what would be the best solutions to overcome them. I recently managed to find the time and right idea to build one. In this post, I'd like to highlight my key takeaways and design principles that I put into the project.

## Introducing "the project"
There is not much to gain from a toy-project, it will end up sitting on a hard drive collecting dust. So, how can one fully experience all the consequences of the technical choices with a toy project?

I came up with a project that can potentially generate profit or at least could be sustainable to operate.  There is no shortage of URL shorteners out there, therefore I decided to build a [very niche one for Amazon Affiliates called aCart.to](https://acart.to). 

It works like a regular shortener, but as input it takes a list of Product IDs instead of URL. Based on the list, the system pre-builds and stores URL that:

- redirects to the amazon cart page with products pre-added
- could link to multiple products

After careful consideration, I’ve highlighted the 3 most important things to consider in a project like this.

## Pick a response code wisely
If the browser requests a URL from an external server, it would receive a [response code](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes) with a requested page. When using a URL shortener, page content will not be returned, but a redirect request will be issued with the following _response code_.


If a shortener will be used by others, it's crucial to have a __301 response code__. If you’re building a shortener for internal use - a __302 response code__ could be more appropriate. For any technically savvy user this would be the biggest criteria for a choice. So it's important to understand the difference to make the right call.
### 301 redirect
A 301 redirect says that the URL requested (_short URL_) has “permanently” moved to the _long URL_. Since it’s a permanent redirect, search engines finding links to the short URLs will credit all those links to the _long URL_.
### 302 redirect
302 redirect is a “temporary” one. If that’s issued, search engines assume that the _short URL_ is the “real” URL and just temporarily being pointed elsewhere. That means link credit does not get passed on to the _long URL_.
### Other
Some URL shorteners on a market are very creative and manage to use 200 or 303 codes while redirecting requests. I’m not completely aware of the motivations for doing so, but it seems “shady” and I would personally recommend against such a practice.
## Keep it (really) short!
Since the biggest use-case for shorteners are social networks with limited character capacity per message, then it's really important to lose all the fat from a short URL.
### Aim for a short domain
The first step in the right direction should be your domain name - it should be as short as possible. Keep in mind that finding short  `.com` or `.org` domains will be costly, so you might want to look for [tld domains of other countries] (https://en.wikipedia.org/wiki/List_of_Internet_top-level_domains#Country_code_top-level_domains). (`.ai` sound really cool!) These domains come with a SEO penalty and you’d be less likely to rank in google, but they are short. That’s the tradeoff most shorteners go with.

But there is even more fat we can lose here.
### Lose www, but don't lose performance and security
A lot of people argue that "www.domain.com" vs "domain.com" usage in domains is merely a cosmetic difference. But unfortunately, dropping `www` from URL can have dire consequences because of how DNS records work.

Unfortunately, most DNS providers work only with two domain record types - _A record_ or _CNAME record_.

- *CNAME record* - requires that it be the only record for that domain
- *A record* - can only point to a IP address

In most cases, to use domains with no ‘www’, you’re required to use A-record with a pointer to the DNS provider's load balancer. In case of DDoS attack on the DNS provider, your domain will be very likely to stop answering.

_A record_ also comes with a performance hit, requests for _CNAME record_ could be served by a closest server. Requests for _A record_ always get resolved through one instance, it will probably be the most loaded instance available.

Netlify goes into great lengths to explain this problem in a blog post titled [“To www or not www”](https://www.netlify.com/blog/2017/02/28/to-www-or-not-www/).

My solution was to move the domain over to a DNS provider that could handle __ALIAS records__ or __CNAME Flattering__. In my case, I had to move from [namecheap](https://namecheap.com) to [dnsimple](https://dnsimple.com). Support for this feature usually costs an additional monthly fee. 
### Bijective function for “hashing”
The shortener has to come up with a public ID that uniquely identifies shortened URLs. In a lot of cases, clients will rule against your server if you're stably generating a long hash for a public ID.
 
I used a bijective function that could transform a number into a hashed text and inverse the text back into a number. Thanks to the internet, it wasn’t that hard to find an [example on stackoverflow](https://stackoverflow.com/questions/742013/how-do-i-create-a-url-shortener/742047#742047).

This function significantly simplified table design in the database. I’m able to convert the primary key to a hash text and pinpoint exact records in a table based on hash text. Not having other indexes in a table is a great speed improvement on it’s own. 
## Don't lose referrals
HTTP Referrer is an option in the HTTP header field that stores the address of the webpage that this request originated from.

There is a very thin line between a URL shortener service and a referrer cloaking service (like [nullrefer](http://www.nullrefer.com)). And this thin line is called "Pass HTTP Referrer". You probably don't want to unintentionally become a cloaking service.

Passing referrer information along to a long URL in ruby is just one line of code, but a really important line of code.

```ruby
headers["HTTP_REFERER"] = request.referrer if request.referrer
```

Unfortunately, there are a number of cases when the HTTP_REFERER parameter will be empty or could not be passed. Due to specification, HTTP_REFERER will not be passed if a long URL uses insecure protocol (http://). 

> If a website is accessed from a HTTP Secure (HTTPS) connection and a link points to anywhere except another secure location, then the referer field is not sent.

But nowadays we have [Referrer Policy](https://w3c.github.io/webappsec-referrer-policy/) as a solution that could partially mitigate these issues by just adding meta tag to HTML pages.

```
<meta name="referrer" content="origin">
```
## Summary
So, when it comes to launching a URL shortener service, keep in mind that these three issues are the biggest you need to be aware of. A lot of other articles explain how to write this service, but don’t give you a project wide overview of launching it into the public.

I hope you found this article useful. If you need a hand in thinking through some of your existing projects - I’m also available for hire!
