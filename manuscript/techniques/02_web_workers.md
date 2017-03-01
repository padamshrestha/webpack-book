# Web Workers

[Web workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) provide a way to push work outside of main execution thread of JavaScript making them convenient for long running computations and background work.

Moving data between the main thread and the worker comes with communication-related overhead. The split provides isolation that forces workers to focus on logic only as they cannot manipulate the user interface directly.

The idea of workers is useful on a more general level. [parallel-webpack](https://www.npmjs.com/package/parallel-webpack) uses [worker-farm](https://www.npmjs.com/package/worker-farm) underneath to parallelize webpack execution.

As discussed in the *Build Targets* chapter, webpack allows you to build your application as a worker itself. To get the idea of web workers better, I'll show you how to build a small worker using [worker-loader](https://www.npmjs.com/package/worker-loader).

## Setting Up Worker Loader

To get started, install *worker-loader* to the project:

```bash
npm install worker-loader --save-dev
```

To keep it easy to implement, we'll tell webpack which files should be treated as loaders through inline loader definitions. Another option would be to push it to webpack configuration as discussed in the *Loader Definitions* chapter.

In that case, you have to filter the files somehow. One option would be to rely on a naming convention so that you name each worker with a `.worker.js` suffix. You could also push them to a directory as that's enough context information too.

## Setting Up a Worker

A worker has to do two things: listen to messages and respond. Between those two actions, it can perform a computation. In this case, we'll accept text data, append it to itself, and send the result.

**app/worker.js**

```javascript
self.onmessage = ({ data: { text } }) => {
  // Append the given text to itself to prove
  // the worker does its thing.
  self.postMessage({ text: text + text });
};
```

## Setting Up a Host

The host has to instantiate the worker and the communicate with it. The idea is almost the same except the host has the control:

**app/component.js**

```javascript
import Worker from 'worker-loader!./worker';

export default function () {
  const element = document.createElement('h1');

  const worker = new Worker();
  const state = {
    text: 'foo',
  };

  worker.addEventListener(
    'message',
    ({ data: { text } }) => {
      state.text = text;

      element.innerHTML = text;
    }
  );

  element.innerHTML = state.text;
  element.onclick = () => {
    worker.postMessage({ text: state.text });
  };

  return element;
}
```

After you have these two set up, it should work. As you click the text, it should mutate the application state as the worker completes its execution. To demonstrate the asynchronous nature of workers, you could try adding delay to the answer and see what happens.

T> [webworkify-webpack](https://www.npmjs.com/package/webworkify-webpack) is an alternative to *worker-loader*. The API allows you to use the worker as a regular JavaScript module as well given you avoid the `self` requirement visible in the example solution.

## Conclusion

The important thing to note is that the worker cannot access the DOM. You can perform computation and queries in a worker, but it cannot manipulate the user interface directly.

To recap:

* Web workers allow you to push work out of the main thread of the browser. This separation is useful especially if performance is an issue.
* Web workers cannot manipulate the DOM. Instead, it is best to use them for long running computations and requests.
* The isolation provided by web workers can be used for architectural benefit. It will force the programmers to stay within a specific sandbox.
* Communicating with web workers comes with an overhead that may make them less useful. As the specification evolves, this may change.

I will discuss the topic of internationalization in the next chapter.
