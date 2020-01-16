---
title: Progressive Mailchimp sign-up form with Netlify
layout: post
description: building progressive dynamic forms with JAMstack on Netlify
---

_This article describes solution for JAMstack website, that usually requires custom back-end. I'll give more details and code to build a solid "works-at-all-times" sign-up form with a custom serverless function_

# Background

I'm currently running a [curated newsletter for amazon sellers](https://www.fbamonthly.com). I've been looking for a similar service - found non. So I started it, mostly to build audience I was interested in serving.

Almost one year in... with a very basic service mash-up of [Carrd](https://carrd.co/) and [Mailchimp](https://www.mailchimp.com) we managed to grow with average 70 subscribers a month (with not much marketing and zero budget) and some sponsorship content.

I've been thinking of growing this newsletter beyond email to increase our reach, so to say.. I wrote down some requirements for version 2 of this project and decided to build a JAMstack website. 

There have been a lot of _nice to have's_, but main requirements have been the following:

- Bulletproof Mailchimp subscription that works even with Javascript disabled,  operability with aggressive ad-blockers and foolproof (yeah, I qualify as one)
- Contact Form has been our money making machine, it should work flawlessly
- All the assets and content hosted on our domain - to get all the SEO benefits
- RSS feed for people to consume our content
- Low maintenance and low cost

I will focus specifically on __contact form__ and __subscription form__ in this article. For a lot of people, this seemed like non-trivial functionality to get working with a static website. 

# Netlify
I was already using [Netlify](https://www.netlify.com) for a couple of projects and experience was great. So I found no reason to look for anything else, but you might want to explore other options -- JAMstack services are booming. Most of ideas I describe here are easily transferable to other platforms.


In Netlify case, it offers a lot of useful building blocks for a static website. With time, I managed to find some rough edges and edges cases that don't work. But for a lot of projects Netlify offers a complete solution of nicely integrated and easy to use functionality. And you are always free to pick any other third party that suits you better or even roll with your own.

__Contact form__ was really easy to implement with [Netlify Forms](https://www.netlify.com/docs/form-handling/). It took me less than 5 minutes to have contact form with basic spam filters up an running. It was amazing, but will not work for a mailchimp subscription form in a way I'd wanted it. I can [hookup serverless functions to forms](https://www.netlify.com/docs/form-handling/#serverless-functions-integration), but there is no sane way to validate input and make sure to have mailchimp keys checked.

So, here is where [Netlify functions](https://functions.netlify.com/) come in play..

# Netlify Functions
To make a short analogy to explain - everything you heard about AWS Lambda is true for Netlify Functions. But list of supported languages is shorter, only Go and JavaScript. I'd love it if they had Ruby support, but Javascript is good enough too.

I've looked around to find any suitable ready-made code, but found examples that didn't fit my initial requirements. Which are:
- Handle regular form submission
- Handle AJAX request
- Validate that environments with mailchimp keys have been set properly
- Have a proper error handling for input field

So, I set out to build my own function and will drive you through implementation I ended up adopting.


# Implementation

_If you're here for code - check out [my public gist](https://gist.github.com/skatkov/b524a6e60a5313acc4d299471a2a3902) or read further for comments_

## 1. Implementing a form submission
This is a very basic HTML form. But there are to important parts to note: 
1. Endpoint (__/.netlify/functions/form-handler__) I used for form submission - is a function that we will create shortly
2. div element with id "message" is our placeholder for success messages

```html
<div id="message"></div>
<form id="newsletter" class="subscribe" action="/.netlify/functions/form-handler" method="post">
  <input type="email" id="inputEmail" name='email' placeholder="Enter email to subscribe for FREE" class="email" required autofocus>
  <button class="button" type="submit">Subscribe</button>
</form>
```

## 2. Implementing an AJAX request
This part is really horrible -- I'm not using TypeScript or jQuery, it's just standard JavaScript and it works everywhere.

Again, two important parts to notice:
- AJAX request is executed against same endpoint that handles form submission
- JavaScript uses __id="message"__ element for success message

```html
<script>
  var form = document.getElementById('newsletter');

  form.addEventListener("submit", function(e) {
    e.preventDefault();
    email = document.getElementById('inputEmail').value;
    submitEmail(email)
  });

  function submitEmail(email) {
    fetch('/.netlify/functions/form-handler', {
        method: 'post',
        body: JSON.stringify({
          email: email
        })
      }).then(function(response) {
        return response.json();
      }).then(function(data) {
        messageDiv = document.getElementById('message');
        messageDiv.innerText = 'Confirmation email has been sent!'
      });
  };
</script>
```

## 3. Setting up environment for functions
I ended-up directly using only 3 JavaScript libraries (and shit ton of transitional dependencies that go with them) to make this function work:
- __netlify-lambda__ so we can test lambda functions locally
- __dotenv__ library so we can store all mailchimp keys in ENV variables
- __axios__ to make to do calls to mailchimp API

I've added these lines to _packages.json_ file (including some convenience methods to get netlify-lambda working locally)

```json
{
  "scripts": {
    "start": "netlify-lambda serve source/lambda",
    "build": "netlify-lambda build source/lambda"
  },
  "dependencies": {
    "axios": "^0.19.0",
    "dotenv": "^8.1.0",
    "netlify-lambda": "^1.6.3"
  }
}

```

## 4. Building a netlify function

There are some important elements here to notice:
- To support both AJAX and form submission we have to deal with parsing body of request. __querystring.parse__ is for form data and __JSON.parse__ for AJAX request.
- In case if it's form submission ( determined by looking up 'content-type' value), then redirect to /thanks.html will be issued. In case of AJAX plain json is returned.
- Before doing any calls to mailchimp - we valid all data we have to deal with (input and env variables).
- I've spent also quite some time figuring how to present errors, that API could return.

Other then that, this seems like a straight forward method that passes some data to 3-rd party API.

```javascript
import { parse } from 'querystring'
const axios = require('axios');
const mailChimpAPI = process.env.MAILCHIMP_API_KEY;
const mailChimpListID = process.env.MAILCHIMP_LIST_ID;


exports.handler = (event, context, callback) => {
  let body = {}
  console.log(event)
  try {
    body = JSON.parse(event.body)
  } catch (e) {
    body = parse(event.body)
  }

  if (!body.email) {
    console.log('missing email')
    return callback(null, {
      statusCode: 400,
      body: JSON.stringify({
        error: 'missing email'
      })
    })
  }

  if (!mailChimpAPI) {
    console.log('missing mailChimpAPI key')
    return callback(null, {
      statusCode: 400,
      body: JSON.stringify({
        error: 'missing mailChimpAPI key'
      })
    })
  }

  if (!mailChimpListID) {
    console.log('missing mailChimpListID key')
    return callback(null, {
      statusCode: 400,
      body: JSON.stringify({
        error: 'missing mailChimpListID key'
      })
    })
  }

  const data = {
    email_address: body.email,
    status: "pending",
    merge_fields: {}
  };

  const subscriber = JSON.stringify(data);
  console.log("Sending data to mailchimp", subscriber);

  // Subscribe an email

  axios(
    {
      method: 'post',
      url: `https://us19.api.mailchimp.com/3.0/lists/${mailChimpListID}/members/`, //change region (us19) based on last values of ListId.
      data: subscriber,
      auth: {
        username: 'apikey', // any value will work 
        password: mailChimpAPI
      }
    }
  ).then(function(response){
    console.log(`status:${response.status}` )
    console.log(`data:${response.data}` )
    console.log(`headers:${response.headers}` )

    if (response.headers['content-type'] === 'application/x-www-form-urlencoded') {
      // Do redirect for non JS enabled browsers
      return callback(null, {
        statusCode: 302,
        headers: {
          Location: '/thanks.html',
          'Cache-Control': 'no-cache',
        },
        body: JSON.stringify({})
      });
    }

    // Return data to AJAX request
    return callback(null, {
      statusCode: 200,
      body: JSON.stringify({ emailAdded: true })
    })
  }).catch(function(error) {
    if (error.response) {
      // The request was made and the server responded with a status code
      // that falls out of the range of 2xx
      console.log(error.response.data);
      console.log(error.response.status);
      console.log(error.response.headers);
    } else if (error.request) {
      // The request was made but no response was received
      console.log(error.request);
    } else {
      // Something happened in setting up the request that triggered an Error
      console.log('Error', error.message);
    }
    console.log(error.config);
  });
};

```

# Conclusions
This functionality is live - so it's a good indication that I'm quite satisfied and confident with solution. It's an amazing experience to get so far with a static website. It just works (really fast!), it's cheap and it's easy to scale.

This PageSpeed Insight result has been completely unintended benefit of JAMstack.

![Alt Text](https://thepracticaldev.s3.amazonaws.com/i/q1g1xio6oedmzs7g3t0h.png)

But there are issues in this code I didn't anticipated. Later I hope to address these:

- Using __axios__ gem has made it almost impossible to test mailchimp subscription on a localhost, due to Cross Origin issues. It seems that I can resolve these issues once I switch to __request__ library. 
- Error handling through console.log bothers me quite a lot. It would be nice to have it in my error catcher of choice, in my case [Honeybadger should work](https://docs.honeybadger.io/lib/node.html#honeybadger-lambdahandler-handler-for-aws-lambda).
- Serverless function is so specialized, and still > 100 lines of code seems like a lot. Some refactoring could be appropriate :)


Thanks for reading! 

Did anyone built side-projects with JAMstack sites? How was your experience?