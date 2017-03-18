---
layout: post
title:  "How I Discovered the Cause of Slowness in My Express App"
date: 2017-03-18T21:30:00+08:00
categories: hacking
tags:
- Express
- Security
- Debugging
---

Recently I discovered some slowness in my express app.

A bit background here: in the platform we are building here in [Madadata](http://www.madadata.com), we are using an external service to provide user authentication and registration. But in order to test against its API without incurring the pain of requesting from California (where our CI servers are) to Shanghai (where our service providers' servers are), I wrote a simple [fake version of their API service](https://github.com/Jimexist/fake-leancloud-auth)  using Express and Mongoose.

We didn't realize the latency of my service until our recently started load testing, where it shows that more than half of requests didn't return within 1 second and thus failing the load test. As a simple Express app using Mongoose there is hardly any chance of getting it wrong, at least not anywhere near 1 second of latency.

![v0.4.0](/assets/2017/03/v0.4.0.png)

The screenshot above for running [mocha](https://mochajs.org/) test locally revealed that there is indeed some problem with the API service!

## What went wrong?

From the screenshot I can tell that not all APIs are slow: the one where users log out and also the one showing current profile is reasonably fast. Also, judging from the dev logs that I printed out using [morgan](https://www.npmjs.com/package/morgan), for the slow APIs, their response time collected by Express is indeed showing a consistent level of slowness, (i.e. for the red flagged ones, you are seeing a roughly sum of latency of two requests above them, respectively).

This actually rules out the possibility that the slowness comes from *connection*, rather than within Express. So my next step is to look at my Express app. (N.B. this is actually something worth ruling out first, and I personally suggest trying at one or two other tools rather than `mocha`, e.g. `curl` and even `nc` before moving on, because they almost always prove to be more reliable than the test code you wrote).

## Inside Express

Express is a great framework when it comes to web server in Node and it has come a long way in terms of speed and reliability. I thought it is more likely due to the plugins and middlewares that I used with Express.

In order to use MongoDB as session store I used [connect-mongo](https://github.com/jdesboeufs/connect-mongo) for backing my [express-session](https://github.com/expressjs/session). I also used the same MongoDB instance as my primary credential and profile store (because why not? it is a service for CI testing after all). For that I used [Mongoose](https://github.com/Automattic/mongoose) for ODM.

At first I suspected that it might be because of the [built-in Promise library](http://mongoosejs.com/docs/promises.html) shipped by default in Mongoose. But after changing it with ES6 built-in one the problem wasn't solved.

Then I figured it is worth to check the schema serialization and validation part. There is only one model and it is fairly simple and straightforward:

```js
const mongoose = require('mongoose')
const Schema = mongoose.Schema
const isEmail = require('validator/lib/isEmail')
const isNumeric = require('validator/lib/isNumeric')
const passportLocalMongoose = require('passport-local-mongoose')

mongoose.Promise = Promise

const User = new Schema({
  email: {
    type: String,
    required: true,
    validate: {
      validator: isEmail
    },
    message: '{VALUE} 不是一个合法的 email 地址'
  },
  phone: {
    type: String,
    required: true,
    validate: {
      validator: isNumeric
    }
  },
  emailVerified: {
    type: Boolean,
    default: false
  },
  mobilePhoneVerified: {
    type: Boolean,
    default: false
  },
  turbineUserId: {
    type: String
  }
}, {
  timestamps: true
})

User.virtual('objectId').get(function () {
  return this._id
})

const fields = {
  objectId: 1,
  username: 1,
  email: 1,
  phone: 1,
  turbineUserId: 1
}

User.plugin(passportLocalMongoose, {
  usernameField: 'username',
  usernameUnique: true,
  usernameQueryFields: ['objectId', 'email'],
  selectFields: fields
})

module.exports = mongoose.model('User', User)
```

## Mongoose Hooks

Mongoose has this [nice feature](http://mongoosejs.com/docs/middleware.html#pre) where you can use `pre-` and `post-` hooks to interact and investigate document validation and saving process.

Using `console.time` and `console.timeEnd` we can actually measure the time spent during these processes.

```js
User.pre('init', function (next) {
  console.time('init')
  next()
})
User.pre('validate', function (next) {
  console.time('validate')
  next()
})
User.pre('save', function (next) {
  console.time('save')
  next()
})
User.pre('remove', function (next) {
  console.time('remove')
  next()
})
User.post('init', function () {
  console.timeEnd('init')
})
User.post('validate', function () {
  console.timeEnd('validate')
})
User.post('save', function () {
  console.timeEnd('save')
})
User.post('remove', function () {
  console.timeEnd('remove')
})
```

and then we are getting this more detailed information in mocha run:

![pre-post](/assets/2017/03/pre-post.png)

Apparently document validation and saving doesn't take up large chunks of latency at all. It also rules out the likelihood a) that the slowness comes from *connection problem* between our Express app and MongoDB server, or b) that the *MongoDB server itself* is running slow.

## Passport + Mongoose

Turning my focus away from Mongoose itself, I start to look at the passport plugin that I used: [passport-local-mongoose](https://github.com/saintedlama/passport-local-mongoose).

The name is a big long but it basically tells what it does. It adapts Mongoose as a local strategy for [passport](https://github.com/jaredhanson/passport), which does session management and registering and login boilerplate.

The library is fairly small and simple, so I start to directly edit the `index.js` file within my `node_modules/` folder. Since function `#register(user, password, cb)` calls function `#setPassword(password, cb)`, i.e. specifically [this line](https://github.com/saintedlama/passport-local-mongoose/blob/0b5da93def0244a551188263bf473d48f3b95876/index.js#L86).

After adding some more `console.time` and `console.timeEnd` I confirmed that the latency is mostly due to this function call:

```js
pbkdf2(password, salt, function(pbkdf2Err, hashRaw) {
  // omit
}
```

## PBKDF2

[The name itself](https://en.wikipedia.org/wiki/PBKDF2) suggested that it is a call to cryptography library. And a second look at [the README](https://github.com/saintedlama/passport-local-mongoose#options) show that the library is using `25,000` iterations.

Like [bcrypt](https://github.com/kelektiv/node.bcrypt.js), `pbkdf2` is also a *slow hashing* algorithm, meaning that it is intended to be slow, and that slowness is adjustable given number of iterations, in order to adapt against ever-increasing computation power. This concept is called [key stretching](https://en.wikipedia.org/wiki/Key_stretching).

As written in the Wiki, the initial proposed iteration number was `1,000` when it first came out, and some recent updates on this number reached as hight as `100,000`. So in fact the default `25,000` was reasonable.

After reducing the iterations to `1,000`, my mocha test output now looks like:

![iter-1000](/assets/2017/03/iter-1000.png)

and finally it is much acceptable in terms of latency and security, for a test application after all!

## Final thoughts

I thought it would be meaningful to share some of my debugging experience on this, and I'm glad that it wasn't due to an actual *bug* (right, a *feature* in disguise).

Another point worth mentioning is that for developers who are not experts on computer security or cryptography, it is usually a good idea not to *homemake* some code related to session/key/token management. Using good open source libraries like passport to start with is a better idea.

And as always, you'll never know what kind of rabbit hole you'll run into while debugging a web server - this is really the fun part of it! 😁
