---
title: Phoenix reborn - open source crypto-exchange Peatio
description: 
layout: post
---
![Crypto exchange](https://thepracticaldev.s3.amazonaws.com/i/4juecuzy6jqzcfo8vva3.png)


I love helping out companies with ruby in their stack. And Upwork helps me find new leads to do that. I'm a regular in the Jobs section and now I started noticing some trends - [Peatio](https://github.com/peatio/peatio) comes up often. But after I inspected the open source code, it seems like too much work to get it production ready:

- The original Peatio project is widely criticised as abandon-ware
- Other forks I've seen address a few issues, but soon become unmaintained as well. 

I never took those jobs -- budget/plan always seemed inappropriate. 

In the not that distant past a friend approached me to get another fork working - [rubykube](https://github.com/rubykube/). It was surprising to find that it was under active development... my Pull Requests got fast feedback and turn-around in overall was quick.

Rubykube fork seemed impressive. So I agreed to help!

## Rubykube
I personally applaud any company that tries to make a living out of an open source project. It's an extremely complicated thing to monetize -- we all heard of success stories, but they are rare compared all the failure stories.

I accidentally found a job posting of theirs on Upwork - here is how they describe their own company for potential new hires:


> We're a French start up with a little more than 60 full time developers and a remote team of another 30-40 team members. We support a Telegram community of talented developers who believe that our product is the most innovative solution in our niche market 

Read [full work proposal](https://www.upwork.com/jobs/_~01a910275c4ab11cd4/) and apply if your interested.

I like to believe, that companies are less likely to lie about their company for potential new hires. So this description is probably the closes to reality (compared to their marketing material).

It seems like company is [heavily invested](https://www.pxn.one/) in this project. And having ~100 employees and growing - seems like big and stable ship, that doesn't plan to slow down anytime soon. 

## Changes in fork

### Biggest change
The original Peatio project was a monolithic rails application. I will not get into a debate about how bad or good it is, but rubykube fork moved away from that and to a micro-services architecture. This is a pretty reasonable trade-off -- it is going to be much more complicated to deploy/monitor all those services but there are significant gains in this as well.

- Security-wise it would be easier to isolate some of the components from public access
- It's much easier to customize interface and keep up to date with upstream, since every service has an API.
- There is a common assumption that it's easier to scale


### Events API

It was interesting to see pusher dependency being removed from the project. It's being replaced by Event Source's pattern with Events API and RabbitMQ. 

Event Sourcing is a trendy keyword in Ruby community. It's sooo-oo popular, that there is a whole conference dedicated to discussing Event Sourcing + Ruby/Rails.

I too believe, that this is safest way to scale.

### Plugins

The Rubykube devs have added plugins functionality. So far there are no plugins at sight, but I feel that it's very reasonable to move some optional dependencies to plugins - like sentry (error catcher).

### Crypto Wallet
Look out for upcoming 0.19 release -- it greatly improves crypto wallets support. A lot of issues proposing that things will be improving even further. [Here is a sneak peak from release notes](https://github.com/rubykube/peatio/pull/1538/commits/5a2a42dcb1d211d860358b8c67ce12b78bea30da):

>    1. Multi Wallet support.
>    2. New Blockchain synchronization mechanism.
>    3. Full ERC20 tokens support.
>    4. Spliting of Blockchain read and write Services and Clients.


## Getting started

First of all, there is a [very active telegram community](https://t.me/peatio). Be sure to join it! 

If you want to play around with project locally, it's recommended to use [minimalist local environment](https://github.com/rubykube/peatio#minimalistic-local-development-environment-with-docker-compose). I went straight for [workbench](https://github.com/rubykube/workbench), it runs all the components of the rubykube fork in a Docker.

There is Kubernetics based setup for production, but it seems to be aimed at Google Cloud Platform and no official support for AWS is present(not yeat at least). Per Denys Tun quote in email:

> I will add, that at this time, we do not have the resources to diversify our focus to AWS and will continue working on improving GCP deployment infrastructure. AWS will be on our topic list after version 2.0 of the RKCP stack. I assume another 2-3 months.. 

We managed to get it running on AWS with convox.com. This required couple of workarounds, but it works. So my guess, that AWS will be officially supported soon enough. It would be fantastic if RabbitMQ could be replaced with SQS on AWS... I understand that it's not a drop-in replacement probably, but... *wink*


## So what would it take to build your own exchange?
Quote from rubykube:
> "You may not need to have an active developer on Peatio source code, however, we recommend the following team setup: 1 dev/ops, 3 frontend developers (react / angular), 2 QA engineers, 1 Security Officer."

I mostly agree with this recommendation. BUT... If we would roughly estimate.. It would probably take 3 months to get a first running version. After that, it's ongoing costs -- at least one dev/ops to keep lights running? Doing regular security audits? I don't have answers to those questions.

Why do you need so much front-end engineers? It's easy, all the back-end is being written by rubykube. It goes with very basic landing/exchange pages and [morally and technically outdated trading-ui](https://github.com/rubykube/peatio-trading-ui). It's expected to be replaced with a modern client facing application, that will communicate with peatio through API calls ( [preferably through react or preact](https://github.com/rubykube/peatio-react) ).

In a perfect world, this should be a 3 point plan:
- Get peatio running
- Start adding a lot of coins
- Get your own front-end

But don't expect things to be easy, world is not perfect. It's not something you can outsource easily or get a cheap deal on. If you don't have technical person you can trust this with, [hire rubykube](http://peatio.tech/).


## Skin in game
I didn't made a call if rubykube fork is worthy for a serious exchange project, someone else did. But: 
- It's the best fork of peatio project I've seen. 
- While I could argue about some implementation details, I generally agree with forks direction. 
- Things are ramping up really fast, very stable release cycle.

For anyone wanting to get involved in this project I would strongly recommend to contribute back. I had a welcoming experience -- company offered significant discounts to their service if we're planning to dedicate our resources to project. Project lead offered a possible governance model with people contributing have a vote in projects future.