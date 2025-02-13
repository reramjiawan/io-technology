---
title: 'Making Web Workers enjoyable'
date: '2024-07-05'
tags: ['frontend', 'web workers', 'javascript', 'typescript']
images: ['/articles/making-web-workers-enjoyable/circuit-board-6557683_1280.jpg']
summary: 'In a single threaded environment Web Workers allow for offloading intensive tasks to keep the main thread free and responsive.'
authors: ['ravindre-ramjiawan']
theme: 'blue'
---

## Table of Contents

<TOCInline toc={props.toc} exclude={["Table of Contents"]} toHeading={2} />

## Background

In the dynamic and ever-evolving landscape of web development, developers universally strive to create responsive applications.
The goal is to deliver a seamless user experience, maintaining a minimum of 60 frames per second, a benchmark set by the highly responsive[^1] mobile apps that users have become accustomed to.
Furthermore, unhandled user input can lead to a subpar user experience, underscoring the importance of comprehensive input handling in the quest for optimal responsiveness.
However, this can often present a significant challenge due to the single-threaded[^2] nature of JavaScript.

In the current era, with an abundance of frontend libraries at our disposal, it’s increasingly easy to compromise on performance and responsiveness as these libraries consume substantial resources.
This implies that when JavaScript is tasked with heavy computations or data processing, it can result in an unresponsive or janky user interface (UI), leading to user dissatisfaction.
Therefore, it is crucial to manage resources effectively to ensure a smooth and responsive UI.

## Web Workers

Web Workers[^3] were first published by the World Wide Web Consortium (W3C)[^17] and the Web Hypertext Application Technology Working Group (WHATWG)[^18] on April 3, 2009.
The W3C and the WHATWG conceptualize web workers as scripts that operate continuously.
These scripts are designed to run without being disrupted by other scripts that react to user interactions such as clicks.
By ensuring these workers are not interrupted by user activities, web pages can maintain their responsiveness while concurrently executing extensive tasks in the background.
This approach allows for a smoother and more efficient user experience.

Web Workers are not to be confused with Service Workers[^19]. Service Workers serve a different purpose and act as proxy server to enable offline experiences by intercepting network requests.
Service Workers also run in a worker context meaning that they run in a separate thread.

### How to create a Web Worker

To make use of a Web Worker you have to create a new Worker by calling the `Worker()` constructor and passing the URL of a script file that will be executed in the Worker thread.

```typescript
// Vanilla
const worker = new Worker('heavy-calculation-script.js')
```

Nowadays with the usage of web bundlers such as Webpack[^4] or Vite[^5] Web Workers require a relative module url.

```typescript
// Bundlers
const worker = new Worker(new URL('./heavy-calcluation-script.js', import.meta.url))
```

### How to communicate with a Web Worker

Once you have created your Web Worker you are now able to send and receive messages from it.
To send messages you make use of the `postMessage()` method and to listen for messages you can use `addEventListener()` or set a callback directly on the `onmessage` property.

```typescript
const worker = new Worker('heavy-calculation-script.js')

// Sending a message to the Web Worker
worker.postMessage('Hello from main thread!')

// Listening for messages from the Web Worker
worker.onmessage = ({ data }: MessageEvent) => {
  console.log(data) // Logs: Hello from worker!
}

// Using event listeners
worker.addEventListener('message', ({ data }: MessageEvent) => {
  console.log(data) // Logs: Hello from worker!
})
```

You can send anything that is serializable[^6] when using `postMessage()`. JavaScript uses the structured clone algorithm[^7] to perform copying complex objects.

```typescript
// heavy-calculation-script.js Web Worker

// Sending a message to the Main Thread
self.postMessage('Hello from worker!')

self.addEventListener('message', ({ data }: MessageEvent) => {
  console.log(data) // Logs: Hello from main thread!
})
```

In the context of a web worker `self`[^8] refers to an object that points to current Web Worker context. It is a reliable way to reference the worker context, unlike the `this` keyword, which can behave unpredictably in various situations.
Normally when using `this` in the global execution context[^9] will refer to the `Window`[^10] object.

### Drawbacks of Web Worker communication

Whilst the Web Worker API to send and receive messages gets the job done it is not very developer friendly because of its low-level API.
It requires a lot of manual management of message routing and payload marshalling[^11].
There are certain patterns that seem to work nicely with `postMessage()` such as the Flux[^12] pattern but there are better libraries out there that can make the use Web Workers a lot more enjoyable and intuitive.

## Comlink

Comlink[^13] is a tiny library developed by Google. Its primary function is to simplify the process of working with Web Workers by eliminating the complexities associated with using `postMessage()`.
It achieves this by adopting an RPC (Remote Procedure Call)[^14] style for message transmission and leveraging JavaScript Proxies[^15] that maintain a reference to the original target.
In essence, Comlink enables seamless access to any element from the Main Thread within a Web Worker and vice versa.
This bidirectional accessibility significantly enhances the developer experience.
Furthermore, when used together with TypeScript[^16] it supports autocomplete features, making coding even more efficient and enjoyable.

### How to use Comlink

Comlink provides a set of functions that help connect the main thread to the Web Worker and vice versa is fairly straight forward as shown below.
The `wrap()` function wraps the Worker and takes the other end of a Message Channel[^21] as an argument and returns a proxy.
This proxy will have all properties and functions of the exposed value from the other thread.
However, access to these properties and function invocations are inherently asynchronous.
This means that a function that would normally return a number will now return a `Promise()`[^20] for a number.

```typescript
import { wrap } from 'comlink'

// Create Web Worker
const worker = new Worker('heavy-calculation-script.js')

// Wrap the Web Worker using the wrap method from Comlink
const wrappedWorker = wrap(worker)

// Call any exposed methods from your Web Worker
// Since its a Promise you can use either await or then
wrappedWorker.exposedMethod().then(console.log) // Logs: 5
```

The `expose()` method from Comlink is used to make a local object available to the other end of the Message Channel.
It can be viewed as the Comlink equivalent of `export`. This method takes an object and exposes it to the other thread, allowing the other thread to access its properties and methods.

```typescript
// heavy-calculation-script.js Web Worker
import { expose } from 'comlink'

const api = {
  exposedMethod() {
    return 5
  },
}

// Call expose from Comlink to expose anything you like for the Main Thread to have access to
expose(api)
```

## Conclusion

I hope this gives a better understanding on how Web Workers can help with offloading heavy computations, intensive tasks or long-running pieces of code.
This allows for the Main Thread to run as efficient and responsive as possible for not only a better user experience but also a great developer experience when using libraries such as Comlink.

[^1]: https://en.wikipedia.org/wiki/Responsiveness
[^2]: https://en.wikipedia.org/wiki/Thread_(computing)
[^3]: https://en.wikipedia.org/wiki/Web_worker
[^4]: https://webpack.js.org/guides/web-workers/
[^5]: https://v3.vitejs.dev/guide/features.html#web-workers
[^6]: https://en.wikipedia.org/wiki/Serialization
[^7]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm
[^8]: https://developer.mozilla.org/en-US/docs/Web/API/WorkerGlobalScope/self
[^9]: https://en.wikipedia.org/wiki/Scope_(computer_science)
[^10]: https://developer.mozilla.org/en-US/docs/Web/API/Window
[^11]: https://en.wikipedia.org/wiki/Marshalling_(computer_science)
[^12]: https://facebookarchive.github.io/flux/
[^13]: https://github.com/GoogleChromeLabs/comlink
[^14]: https://en.wikipedia.org/wiki/Remote_procedure_call
[^15]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy
[^16]: https://en.wikipedia.org/wiki/TypeScript
[^17]: https://www.w3.org/
[^18]: https://whatwg.org/
[^19]: https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API
[^20]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
[^21]: https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel
