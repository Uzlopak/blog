# Implementing a new feature in a fastifyjs plugin
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
regarding the possibilty to limit the download speed of requests. This feature
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

1. How do you implement a ThrottleStream in nodedjs to controle the bandwith?
2. Is fastify core capable handle the response with a ThrottleStream?

#### ThrottleStream: How can we implement it?

If you have experience with streams, you will immediatly think of 
Transform-Streams. Should be easy.

But even if you have no experience with Streams, you should not give up.
A developer learns everyday a new aspect of his progamming language or the
runtime he uses.

Should you now deepdive into the nodejs documentation about streams?
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

So now we check the fastify documentation. We dont want to change fastify
core if it is not necessary. Fastify provides hooks to modify its behaviour. So 
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

Is it the end of our jourrney? As you see that this blog post is much longer,
you can confidently say no.

But what if we send a readstream? We change our `example-throttle.js` accordingly:

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
    // pass the throttleSteam as payloa
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
downloads? One-Click-Hosters. They dont throttle the output of their usual rest api calls,
but they throttle their file downloads.
We can assume, that devs, who want to throttle downloads, will use it for file downloads.

So our scope regarding what to throttle got clear.


