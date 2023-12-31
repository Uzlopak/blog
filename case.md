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
so we create a new one `example-throttling.js`.

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

The `example-throttling.js` is currently not handling the throttling, but as we
are writing our solution, it will change over time. And in the end it should
be a real example on how to throttle requests. 

We open http://127.0.0.1:3000/ in the browser and see the message:
`{"status":"ok"}`. Now we check the stdout of the nodejs process and see
`{ isStream: false }`. Payload is not a stream. 

Is it the end of our journey? As you see that this blog post is much longer,
you can confidently say no.

But what if we send a ReadStream? We change our `example-throttling.js` accordingly:

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

We open again http://127.0.0.1:3000/ and see the content of the `example-throttling.js`. 
In the stdout of the nodejs process we see `{ isStream: true }`.

Perfect. We know now, that it is in some cases a stream. 

Now we need to install `throttle` by doing an `npm i throttle` and change again the 
`example-throttling.js`. 

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
currently that we can just throttle streams.
This needs a consideration: Is it enough to support throttling of steams? Or how does it fit
in the fastify lifecycle. If we look at the lifecycle of a request, we see that we have a
`preSerialization` lifecycle stage. So the payload we get in the `onSend` hook should
already be validated and enriched with all the data it needs. So if we need to support
strings, Buffers and other non-stream-objects, we can transform them first to a stream,
and then pass them to the `ThrottleStream`. 
In my opinion it would be enough for the first iteration to handle streams. Because, lets be 
honest, in which cases do we actually need throttling? Were in the wild do we see throttled
downloads? One-Click-Hosters and Video-Streaming-Platforms. They don't throttle the output
of their rest api calls, but they throttle their file downloads.
We can assume, that developers will use it for throttling file downloads.

But we know already, that we just need to transform Buffers and strings to streams.

So next iteration of the `example-throttling.js`

```js
'use strict'

const { createReadStream } = require('fs')
const {Readable} = require('stream')
const { resolve } = require('path')
const ThrottleStream = require('throttle')

const fastify = require('fastify')()

fastify.addHook('onSend', (request, reply, payload, done) => {
  if (payload && payload.pipe) {
    const throttleStream = new ThrottleStream({ bps: 1000 })
    payload.pipe(throttleStream)
    done(null, throttleStream)
    return
  } else if (typeof payload === 'string') {
    const throttleStream = new ThrottleStream({ bps: 1000 })
    Readable.from(Buffer.from(payload)).pipe(throttleStream)
    done(null, throttleStream)
    return
  } else if (Buffer.isBuffer(payload)) {
    const throttleStream = new ThrottleStream({ bps: 1000 })
    Readable.from(payload).pipe(throttleStream)
    done(null, throttleStream)
    return
  }
  done(null, payload)
})

fastify.get('/string', (req, reply) => {
  reply.send(Buffer.allocUnsafe(1024 * 1024).toString('ascii'))
})

fastify.get('/buffer', (req, reply) => {
  reply.send(Buffer.allocUnsafe(1024 * 1024))
})

fastify.get('/stream', (req, reply) => {
  reply.send(createReadStream(resolve(__dirname, '../test/fixtures/output.file')))
})

fastify.get('/pojo', (req, reply) => {
  const payload = Array(1000).fill(0).map(v => (Math.random() * 1e6).toString(36))
  reply.send({payload})
})

fastify.listen({ port: 3000 })
```

You can now test each route and see, that we properly throttle on each
route.

We know now that throttling is fully possible. It is actually just a small 
change, so we can implement it already in the first iteration.

So our scope regarding what to throttle is clear.

### More Research: What is feature completeness?

We know now, that throttling is fully possible.
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

So we could implement `ThrottleGroups` in an indirect way.

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

So we should make the ThrottleStream if possible resilient against such cases. 

### Enough Research. Time for Development

#### ThrottleStream: Use package or integrate?

If you have a highly maintained package and a healthy community, it makes sense to use
their packages. But we have here unmaintained packages and basically no progress in features.
Also using existing packages makes sense, when the feature you want to provide has nothing
to do with a http server related technology. Sure. Everything is http related in fastify.
But for example I would not consider connecting to an external service as a core http
technology.
In our case we want to throttle the download speed in a fastify server. We have to ensure
that it works flawless. 

I want to be sure, that everything is under control and doesn't have any side effects.
So I want to integrate the functionality. We can ensure that we have 100% test coverage
and we can manipulate the existing ThrottleStream for our needs.

#### ThrottleStream: Implementing our own solution

There are two approaches for ThrottleStream:
1. Write our own TransformStream with throttling functionality
2. Use the `throttle`-npm as a blue print.

I first tried to implement a custom ThrottleStream.
Programming is not about only copying code and claim it to be your own. We talk here
about computer science, where we want to implement a state-of-the-art-solution.

Unfortunately it was not that easy. After few hours of playing around and writing code,
I realized, that it is not a trivial task. Sure, I could have invested few days, to
implement it and have my own solution. But was it really worth it?

Well, there were some issues I had to tackle. For example, I thought I had a correct
solution just to realize, that the throttle would get faster and faster or slower
and slower. 

Just that you maybe get an idea, what you should do if you have issues like that:
I consulted chatgpt ;)
![image](https://github.com/Uzlopak/blog/assets/5059100/3509b816-deb1-4dfb-bd36-ccf747044625)

![image](https://github.com/Uzlopak/blog/assets/5059100/8da161d1-37cf-434a-9ffe-e245723ea19e)

The explaination was totally correct. But the code it provided to solve that issue
on the other hand was not. 

I wanted to get closer to a solution. So I decided to take the existing `throttle` npm
as a blueprint. 

#### ThrottleStream: Refactoring `throttle`

First things first: We need to first implement tests. Fortunately `throttle` had simple
tests. So I analyzed them and saw that they were written in mocha. So I adapted them
and rewrote them to use `tap`. 

Then I copied the code from `throttle` into the `/lib` folder of the project. `throttle` code
was old, meaning, that it was written for node 6 or 8. For example back in those days,
Buffers where initialized with `new Buffer()`, nowadays you would use `Buffer.alloc()`.  Also
`throttle` is using a npm package called `stream-parser`, which is also written by TooTallNate,
to handle the data processing. Also it uses the `debug` package, which can be replaced
with `debugLog` from nodejs own `util` module.

So with the tests and the code it is possible to refactor. First I integrate the code of
`stream-parser` to the `ThrottleStream` code. Testing reveals alot of uncovered branches.
This indicates it has alot of unnecessary code. We can not just delete uncovered branches,
and be happy, that the test coverage is 100%. You have to actually understand, what the code
is actually doing and if it is not used in any of our use cases, it can be removed.

This process took for me quiete some time. I would estimate the time to be about 12 hours.
When I started the `ThrottleStream` had about 160 LOC from `stream-parser` and about 55 LOC
from `throttle`. When I finished, `ThrottleStream` was about 100 LOC. 

To describe what I did in detail would be imho boring and lengthy. It was constant refactoring.
Remove this or that attribute of `ThrottleStream`, remove methods, integrate code from here to
there, optimize, precalculate values, etc. etc..

#### Tools for refactoring and optimizing code

I love `sonarqube`. Spin up sonarqube and scan the codebase. It reports code smells and
security hotspots. Fixing code smells is useful.

Also the use of `chatgpt` makes in some cases sense.
Ask chatgpt a question and provide the code. You will get detailed insights. But dont trust
the code it provides, it is mostly garbage. For example I asked chatgpt if a 
if-condition is always true, and chatgpt determined yes and explained it. I verified the
explaination and could remove the if-condition.

The tool `deoptigate` is also very nice in analyzing deoptimizations. I would use
the example script and use a small file. Run `deoptigate` with the example and start
`autocannon` on the correct url. After 10 seconds of hitting the server hard,
you stop the `deoptigate` process and it shows you deoptimzations. `ThrottleStream`
had no reported deoptimizations. In my opinion `deoptigate` works best with node 14. 

The tool `0x` is used similar to `deoptigate`, run `0x` with the `-o`-flag and hit
the server with `autocannon`. You should see the hot paths and the optimized
functions. Optimizing the hot paths should be done early, because solving them
later could mean more work.

To some extent you can use `typescript`. Create a `tsconfig.json` in your project,
with the following content:

```json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true,
    "noEmit": true
  }
}
```

Now vscode will use typescript to show potential issues. It works better if you add
jsdoc annotations to your code. But it is far from perfect, e.g. I dont use `class`
but prototypical based inheritance for `ThrottleStream`, and that is not supported
properly by typescript. 

#### Integrating ThrottleSteam into fastify-rate-limit

Now that we have a `ThrottleStream` implementation, it is possible to integrate
it into @fastify/rate-limit. 

##### Typings shape the api contract

I usually start the integration with the typescript definitions. This has the
benefit, that we actually agree an "api contract". We define what options we
expect to activate the plugin. 
Fastify Plugins are usually configured two ways:
1. at plugin registration via the plugin options
2. per route options when registering the route.

Usually the typings of the route options are also used for the typings of the
plugin options. So we should first modify the route options and then check if
the route options are also used by the plugin options.

When designing the api contract, you should keep backwards compatibility for
upcoming features in mind. In my first iteration I want just pass the `bps`
option. For this I will create a new attribute called `throttle` in the options.
`throttle` is an object, where i add the `bps` attribute accordingly. If
we add more feature in the future, we just need to extend the `throttle`
object.
The other option would be adding multiple options to the root of the options
interface. But as you can see, the options interface is already very big.
To manage them is already hard. So if I would add a new option on the root,
I would prefix them with `throttle`, so it would be `throttleBps`. I dont
like it, and in my opinion an object for `throttle` is the better solution.

After we added the typings, we need to add typings tests. Usually we use `tsd`
for typings tests. Basically look for the *.test-d.ts files and add your 
typings test to it. 

#### Integrate ThrottleStream - make it configurable

We have already our solution. It is in our `example-throttling.js`.
We only need to add the `onSend` hook handler to the routeOptions of the routes
accordingly. 

We create a new function called `throttleOnSendHandler` which gets 
`throttleOptions` as a parameter. We take the `onSend` hook handler from our
example return it in the `throttleOnSendHandler` but pass the 
`throttleOptions` to the initialization call of `ThrottleStream`.

We search for `addHook('onRoute')` in the `index.js`. This is the place to
manipulate the options of each route and add the `onSend` hook handler if it is
configured accordingly. 

So basically check if the routeOptions or the pluginOptions have the throttle
options configured and add it to the `onSend` attribute to the routeOptions
of the `onRoute` hook. But keep in mind that the `onSend` option can be 
undefined, a function or an Array of functions.

We create a new function called `addRouteThrottleHook`, which expects
the `routeOptions` of the `onRoute` hook handler and `throttleOptions`
as parameters. We check the value of the `onSend` attribute on the
`routeOptions`. If the `onSend` attribute is undefined, we just assign the
result of `throttleOnSendHandler` to it. If it is a function we create a
new Array with the original value and add the result of the `throttleOnSendHandler`
as second value. And if the value of `onSend` attribute is an Array we
just push the result of the `throttleOnSendHandler` to it.

In the `onRoute` hook handler we extract the `throttleOptions` and call
`addRouteThrottleHook`. 

Thats it. 

Of course we write additional tests to ensure that our solution works. 

#### Documentation

It doesnt matter, how good your implementation is. If the documentation is missing,
your solution is just a blackbox and nobody will use it confidently.

So we edit the Readme.md. 
