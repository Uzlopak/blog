# Implementing a new feature in a fastify plugin
## Case study: Implement ThrottleStream in @fastify/fastify-rate-limit
--------------------------------------------------------------------------------
Implementing a new feature is a daily task for an average developer. Issues
arise when implementing a new feature, which should be used in backend
applications. While a new feature in a backend application is critically
inspected by your colleagues and maybe bugs are reported by the use of your
backend application, implementing a new feature in a widely used community
project has the critical eyes of other maintainers and the necessity to supply
high quality code. Developers, who use the community project, build over time
trust in the code quality and that trust can be shattered if you supply bad
code.

I want to show my approach for implementing a new feature. Maybe you disagree on
some aspects and I am happy to discuss them with you.

### The Feature Request: Limit the download speed of requests

On 29th of April user Uriel29 created an issue in @fastify/fastify-rate-limit,
regarding the possibility to limit the download speed of requests. This feature
was not implemented yet in fastify-rate-limit and mcollina labeled the issue as
`enhancement` and highly appreciated if that solution would be implemented.

Shortly after a user asked for more information and showed interest in
implementing the feature. I don't want to blame the user. Simple tasks are
marked with `good-first-issue`. But it is not a simple task.

Describing a potential approach for a complex task takes usually more time than
implementing it. 

### The Task: Exploring potential approaches and limitations

The first question: Is it (currently) possible? Nothing is more
frustrating to realize after few days of work, that it is not even technically
possible. Less frustrating is of course, if you need to provide a PR to a
dependency. 

Throttling the download speed is a standard feature of other servers, like
nginx. So we should expect that it is potentially possible. 

Nodejs is famous for its streams. So there are two main questions to be
researched: 

1. How do you implement a ThrottleStream in nodejs to control the bandwidth?
2. Is fastify core capable handle the response with a ThrottleStream?

#### ThrottleStream: How can we implement it?

If you have experience with streams, you will immediately think of 
Transform-Streams. Should be easy.

But even if you have no experience with Streams, you should not give up.
A developer learns everyday a new aspect of his programming language or the
runtime he uses.

Should you now deep dive into the nodejs documentation about streams?
Why not. Knowing the technique behind it is key. But in my opinion you will be
not closer to a solution. And you will not be closer to the second question,
regarding fastify.

So after a short moment at the nodejs documentation you should start searching
on npm for potential packages. Keywords should now flow in your mind, and you 
type them into the npm search: `throttle stream` or `bottleneck stream`. You will
find multiple packages, which are potential candidates to come closer to your
question.

You will find packages like `stream-throttle`, `throttle-stream`, `throttle`,
`break` and so on. Developers had already potential solutions to your problem.
But which one will you choose?
Probably sweat will arise on your forehead, seeing that the packages are released
multiple years ago. "Are they still maintained" you will ask yourself. 
The simple answer: It doesn't matter for now. 

I choose [`throttle`](https://www.npmjs.com/package/throttle?activeTab=dependencies) by [TooTallNate](https://github.com/TooTallNate). I know his name from other packages. I assume,
that he will not write garbage - remember: trust is build over time. 
Reading the code, you will realize it is quite simple implementation. Pure
Javascript. No Typescript. This is what we need and the hope that it still works,
after not being maintained for apparently 10 years.

Conclusion: We have found our potential ThrottleStream candidate.

#### Fastify: Can we use ThrottleStream use it?

Till now we did not write any code. Now the time has come to write our first proof
of concept. 

So now we check the fastify documentation. We don't want to change fastify
core if it is not necessary. Fastify provides hooks to modify its behavior. So 
we have to read the documentation about [hooks](https://fastify.dev/docs/latest/Reference/Hooks/).
We need to hook later in the request-lifecycle and have the payload. 
The `onResponse`-hook does not have the payload, but the `onSend`-hook has.
We need to utilize the `onSend`-hook, but is the payload a stream?

To investigate this, the best approach is to check if the package we want to add
the feature contains examples. The examples in fastify-rate-limit are not simple,
so we create a new one `example-thottle.js`.

```js
'use strict'

const fastify = require('fastify')()

fastify.addHook('onSend', (request, reply, payload, done) => {
  // ducktype stream type
  console.log({ isStream: payload && typeof payload.pipe === 'function' })
  done(null, payload)
})

fastify.get('/', (req, reply) => {
  reply.send({ status: 'ok'})
})

fastify.listen({ port: 3000 })
```

The `example-throttle.js` is currently not handling the throttling, but as we
are writing our solution, it will change over time. And in the end it should
be a real example on how to throttle requests. 

We open http://127.0.0.1:3000/ in the browser and see the message:
`{"status":"ok"}`. Now we check the stdout of the nodejs process and see
`{ isStream: false }`. Payload is not a stream. 

Is it the end of our journey? As you see that this blog post is much longer,
you can confidently say no.

But what if we send a ReadStream? We change our `example-throttle.js` accordingly:

```js
'use strict'

const { createReadStream } = require('fs')
const { resolve } = require('path')

const fastify = require('fastify')()

fastify.addHook('onSend', (request, reply, payload, done) => {
  // ducktype stream type
  console.log({ isStream: payload && typeof payload.pipe === 'function' })
  done(null, payload)
})

fastify.get('/', (req, reply) => {
  // for simplicity, we send this file as a read stream
  reply.send(createReadStream(resolve(__dirname, __filename)))
})

fastify.listen({ port: 3000 })
```

We open again http://127.0.0.1:3000/ and see the content of the `example-throttle.js`. 
In the stdout of the nodejs process we see `{ isStream: true }`.

Perfect. We know now, that it is in some cases a stream. 

Now we need to install `throttle` by doing an `npm i throttle` and change again the 
`example-throttle.js`. 

```js
'use strict'

const { createReadStream } = require('fs')
const { resolve } = require('path')
const ThrottleStream = require('throttle')

const fastify = require('fastify')()

fastify.addHook('onSend', (request, reply, payload, done) => {
  if (payload && payload.pipe) {
    // throttle the stream to 1kb/s
    const throttleStream = new ThrottleStream({ bps: 1024 })
    payload.pipe(throttleStream)
    // pass the throttleSteam as payload
    done(null, throttleStream)
    return
  }
  done(null, payload)
})

fastify.get('/', (req, reply) => {
  reply.send(createReadStream(resolve(__dirname, '../test/fixtures/output.file')))
})

fastify.listen({ port: 3000 })
```

In the `test`-folder I create a new `output.file` in a newly created fixtures folder.
I expect that I will use it maybe again in my unit tests. To create it, I run
`truncate --size 100k output.file` and move it in the `test/fixtures` folder. 

We open again http://127.0.0.1:3000/ and because the `output.file` is detected as binary
data it triggers a download. So we we open the download window of the Browse and see following:

![image](https://github.com/Uzlopak/blog/assets/5059100/0345daa9-e5a0-4e29-a489-207f9968069d)

It is successfully throttling the request!

#### Conclusion: Are we there yet?

We have now the knowledge, that throttling of requests is possible. Only limitation is
that we can just throttle streams.
This needs a consideration: Is it enough to support throttling of steams? Or how does it fit
in the fastify lifecycle. If we look at the lifecycle of a request, we see that we have a
`preSerialization` lifecycle stage. So the payload we get in the `onSend` hook should
already be validated and enriched with all the data it needs. So if we need to support
strings, Buffers and other non-stream-objects, we can transform them first to a stream,
and then pass them to the `ThrottleStream`. 
In my opinion it is for the first iteration enough to handle streams. Because, lets be 
honest, in which cases do we actually need throttling? Were in the wild do we see throttled
downloads? One-Click-Hosters and Video-Streaming-Platforms. They don't throttle the output
of their rest api calls, but they throttle their file downloads.
We can assume, that developers, who want to throttle downloads, will use it for file downloads.

So our scope regarding what to throttle got clear.

### More Research: What is feature completeness?

We know now, that throttling is possible and what type of requests we want to throttle.
But we have to take another round of research. Just because we maybe add a feature to throttle,
doesn't mean that other developers will use it. Consumer of plugins have the expectation, that their
use case is already covered or is possible to be realized with a reasonable amount of work.

We need to cover at least the most basic use cases of the feature. The feature needs to
be easily configurable. 

We need to find out the pain points developers have usually with throttling. 

First of all, we open again the npm package and search for the throttle stream packages and
visit the corresponding github/gitlab repository pages and look, which options and/or 
additional functionalities and classes are also provided by the package. Also pull requests
are a good source of information. Especially closed issues and pull requests show tendencies,
about the needs of the users of the package.
And last but not least when checking the forks and the network of each repository can reveal
some useful insights and commits.

I found following relevant issues:
- `ThrottleStream` should handle the change of speed, [[1]](https://github.com/TooTallNate/node-throttle/issues/5) [[4]](https://github.com/TooTallNate/node-throttle/issues/4)
- `ThrottleStream` should have a "burst"-mode, [[2]](https://github.com/TooTallNate/node-throttle/issues/6) [[3]](https://github.com/tjgq/node-stream-throttle/issues/3)
- `ThrottleStream` should be able to have a delay, [[5]](https://github.com/TooTallNate/node-throttle/pull/7), or latency [[6]](https://github.com/tjgq/node-stream-throttle/issues/1)
- `ThrottleStream` should have a "ThrottleGroup", [[7]](https://github.com/tjgq/node-stream-throttle/blob/master/src/throttle.js)

#### Controlling the throttle-speed

The first three issues can be solved by a "easing" functionality. Do you remember jquery-ui
and its [easing functionality](https://jqueryui.com/easing/)? So what we maybe need is to
add an easing option to `ThrottleStream`. We could decide that we extend the `bps` option to
be able to pass a function to it. First parameter is the time since the transfer started and
the second parameter the bytes already sent.

You want a burst mode where you send e.g. 5 Mb in the first 3 seconds and then only 100kb/s? 
Then define a function which exactly does this. 
You want a latency of 1 second? Define a function, which returns 0 for the first second and
then the preferred speed.

#### ThrottleGroups

ThrottleGroups on the other hand, should not be realized in the ThrottleStream itself.
ThrottleStream should not know about other ThrottleStreams but should be controlled by
fastify-rate-limit.

In case of a fastify plugin we have to consider, that a feature will be potentially
used in multiple instances of fastify behind a reverse proxy/load balancer. If so, it has
to be implemented in a resilient way. 

In case of throttling of huge file streams, we can consider the use of download managers.
Of course we can use the original rate limiting feature and allow only 1 request of the 
route. But we should also be able to throttle over multiple fastify instances.

If we can provide a function to the `bps` option, then we can actually set the `bytes`
by loading the value from a store.

So we will implement `ThrottleGroups` in an indirect way.

#### Avoiding unexpected bursts

Also one feature, which I did not find in issues and pull requests, is the fact, that a
user could pause a download while keeping the connection alive. Usually `ThrottleStream`
implementations are time based. A stream starts, sets a `startDate` and then based on the
elapsed time it calculates how much data a user could download since the `startDate`.
This results in a sudden speed burst. 

We have to keep in mind, that a rate-limiting or a speed throttle is a protection against
resource exhaustion. You have a limited amount of traffic and want to provide all users
of your server a good experience, but power users should not be able to degrade the server
performance and thus effecting the experience of normal users.

So we should make the ThrottleStream resilient against such cases. 

### Enough Research. Time for Development

#### ThrottleStream: Use package or integrate?

If you have a highly maintained package and a healthy community, it makes sense to use
their packages. But we have here unmaintained packages and basically no progress in features.

Also I want to be sure, that everything is under control and doesn't have any side effects.
So I tend to integrate the package.
