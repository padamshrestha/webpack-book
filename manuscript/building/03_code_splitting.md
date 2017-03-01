# Code Splitting

Web applications have the tendency to grow big as features are developed. The longer it takes for your application to load, the more frustrating it is to the user. This problem is amplified in a mobile environment where the connections can be slow.

Even though splitting our bundles can help a notch, they are not the only solution, and you may still end up having to download a lot of data. Fortunately, it is possible to do better thanks to **code splitting**. It allows us to load code lazily as we need it.

T> Incidentally, it is possible to implement Google's [PRPL pattern](https://developers.google.com/web/fundamentals/performance/prpl-pattern/) using lazy loading. PRPL (Push, Render, Pre-cache, Lazy-load) has been designed with mobile web in mind and can be implemented using webpack.

## Code Splitting Formats

Code splitting can be done in two primary ways in webpack: through a [dynamic import](https://github.com/tc39/proposal-dynamic-import) or `require.ensure` syntax. We'll be using the former in this project. The syntax isn't in the official specification yet, so it will require minor tweaks especially at ESLint and Babel too if you are using that.

The goal is to end up with a split point that will get loaded on demand. There can be splits inside splits, and you can structure an entire application based on splits.

![Code splitting](images/dynamic.png)

### Dynamic `import`

Dynamic imports look like this:

```javascript
import('./module').then((module) => {...}).catch((error) => {...});
```

The `Promise` based interface allows composition, and you could load multiple resources in parallel like this:

```javascript
Promise.all([
  import('lunr'),
  import('../search_index.json'),
]).then(([lunr, search]) => {
  return {
    index: lunr.Index.load(search.index),
    lines: search.lines,
  };
});
```

It is important to note that this will create separate chunks to a request. If you wanted only one, you would have to define an intermediate module to `import`.

W> The syntax works only with JavaScript after configured the right way. If you use another environment, you will have to use alternatives covered in the following sections.

### `require.ensure`

[require.ensure](https://webpack.js.org/guides/code-splitting-require/#require-ensure-) provides an alternate way:

```javascript
require.ensure(
  // Modules to load, but not execute yet
  ['./load-earlier'],
  () => {
    const loadEarlier = require('./load-earlier');

    // Load later on demand and include to the same chunk
    const module1 = require('./module1');
    const module2 = require('./module2');

    ...
  },
  'optional name'
);
```

As you can see, `require.ensure` definition is more powerful. The problem is that it doesn't support error handling. Often you can achieve what you want through a dynamic `import`, but it's good to know this form exists as well.

T> `require.ensure` supports naming. `require.ensure` blocks that have the same name will be pulled into the same output chunk as showcased by [an official example](https://github.com/webpack/webpack/tree/master/examples/named-chunks).

W> `require.ensure` relies on `Promise`s internally. If you use `require.ensure` with older browsers, remember to shim `Promise` using a polyfill such as [es6-promise](https://www.npmjs.com/package/es6-promise).

### `require.include`

The example above could be rewritten using webpack particular `require.include`:

```javascript
require.ensure(
  [],
  () => {
    require.include('./load-earlier');

    const loadEarlier = require('./load-earlier');

    // Load later on demand and include to the same chunk
    const module1 = require('./module1');
    const module2 = require('./module2');

    ...
  }
);
```

If you had nested `require.ensure` definitions, you could pull a module to the parent chunk using either syntax.

T> The formats respect `output.publicPath` option. You can also use `output.chunkFilename` to shape where they output. Example: `chunkFilename: '[name].js'`.

## Setting Up Code Splitting

To demonstrate the idea of code splitting, we should pick up one of the formats above and integrate it into our project. Dynamic `import` is enough. Before we can implement the webpack side, ESLint needs a slight tweak.

### Configuring ESLint

Given ESLint supports only standard ES6 out of the box, it requires some tweaking to work with dynamic `import`. Install *babel-eslint* parser first:

```bash
npm install babel-eslint --save-dev
```

Tweak ESLint configuration as follows:

**.eslintrc.js**

```javascript
module.exports = {
  ...
leanpub-start-insert
  "parser": "babel-eslint",
leanpub-end-insert
  "parserOptions": {
    "sourceType": "module",
leanpub-start-insert
    "allowImportExportEverywhere": true,
leanpub-end-insert
  },
  ...
}
```

After these changes, ESLint won't complain if we write `import` in the middle of our code.

### Configuring Babel

Given Babel doesn't support the dynamic `import` syntax out of the box, it needs [babel-plugin-syntax-dynamic-import](https://www.npmjs.com/package/babel-plugin-syntax-dynamic-import) to work. Install it first:

```bash
npm install babel-plugin-syntax-dynamic-import --save-dev
```

To connect it with the project, adjust the configuration as follows:

**.babelrc**

```json
{
leanpub-start-insert
  "plugins": ["syntax-dynamic-import"],
leanpub-end-insert
  ...
}
```

### Defining a Split Point Using a Dynamic `import`

A quick way to illustrate the idea might be to set up a module that contains a string that will replace the text of our demo button. Set up a file as follows:

**app/lazy.js**

```javascript
export default 'Hello from lazy';
```

We also need to point the application to this file, so the application knows to load it. An easy way to do this is to bind the loading process to click. Whenever the user happens to click the button, we'll trigger the loading process and replace the content.

**app/component.js**

```javascript
export default function () {
  const element = document.createElement('div');

  element.className = 'fa fa-hand-spock-o fa-1g';
  element.innerHTML = 'Hello world';
leanpub-start-insert
  element.onclick = () => {
    import('./lazy').then((lazy) => {
      element.textContent = lazy.default;
    }).catch((err) => {
      console.error(err);
    });
  };
leanpub-end-insert

  return element;
}
```

If you open up the application (`npm start`) and click the button, you should see the new text in the button.

![Lazy loaded content](images/lazy.png)

The build result is the more interesting part. If you run `npm run build`, you should see something like this:

```bash
Hash: 4f6f78b2fd2c38e8200d
Version: webpack 2.2.1
Time: 2606ms
                    Asset       Size  Chunks             Chunk Names
                   app.js    2.74 kB       1  [emitted]  app
  fontawesome-webfont.eot     166 kB          [emitted]
fontawesome-webfont.woff2    77.2 kB          [emitted]
 fontawesome-webfont.woff      98 kB          [emitted]
  fontawesome-webfont.svg   22 bytes          [emitted]
                 logo.png      77 kB          [emitted]
                     0.js  313 bytes       0  [emitted]
  fontawesome-webfont.ttf     166 kB          [emitted]
                vendor.js     150 kB       2  [emitted]  vendor
                  app.css    3.89 kB       1  [emitted]  app
                 0.js.map  231 bytes       0  [emitted]
               app.js.map    2.78 kB       1  [emitted]  app
              app.css.map   84 bytes       1  [emitted]  app
            vendor.js.map     179 kB       2  [emitted]  vendor
               index.html  274 bytes          [emitted]
   [0] ./~/process/browser.js 5.3 kB {2} [built]
   [3] ./~/react/lib/ReactElement.js 11.2 kB {2} [built]
  [18] ./app/component.js 461 bytes {1} [built]
...
```

That *0.js* is our split point. Examining the file reveals that webpack has wrapped the code in a `webpackJsonp` block and processed the code bit.

### Lazy Loading Styles

Lazy loading can be applied to styling as well. Expand the definition like this:

**app/lazy.js**

```javascript
leanpub-start-insert
import './lazy.css';
leanpub-end-insert

export default 'Hello from lazy';
```

And to have a style definition to load, set up a rule:

**app/lazy.css**

```css
body {
  color: blue;
}
```

The idea is that after *lazy.js* gets loaded, *lazy.css* is applied as well. You can confirm this by running the application (`npm run start`). The same behavior is visible if you build the application (`npm run build`) and examine the output (`0.js`). This is due to our `ExtractTextPlugin` definition.

### Defining a Split Point Using `require.ensure`

It is possible to achieve the same with `require.ensure`. Consider the full example below:

```javascript
export default function () {
  const element = document.createElement('div');

  element.className = 'pure-button';
  element.innerHTML = 'Hello world';
  element.onclick = () => {
    require.ensure([], (require) => {
      element.textContent = require('./lazy').default;
    });
  };

  return element;
}
```

T> [bundle-loader](https://www.npmjs.com/package/bundle-loader) gives similar results, but through a loader interface.

T> The *Dynamic Loading* chapter covers other techniques that come in handy when you have to deal with more dynamic splits.

## Code Splitting in React

The splitting pattern can be wrapped into a React component. Airbnb uses the following solution [as described by Joe Lencioni](https://gist.github.com/lencioni/643a78712337d255f5c031bfc81ca4cf):

```javascript
import React from 'react';

...

// Somewhere in code
<AsyncComponent loader={() => import('./SomeComponent')} />

...

// React wrapper for loading
class AsyncComponent extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      Component: null,
    };
  }

  componentDidMount() {
    // Load the component now
    this.props.loader().then(Component => {
      this.setState({ Component });
    });
  }

  render() {
    const { Component } = this.state;
    const { Placeholder } = this.props;

    if (Component) {
      return <Component {...this.props} />;
    }

    return <Placeholder>
  }
}

AsyncComponent.propTypes = {
  // A loader is a function that should return a Promise.
  loader: PropTypes.func.isRequired,

  // A placeholder to render while waiting completion.
  Placeholder: PropTypes.node.isRequired
};
```

T> [react-async-component](https://www.npmjs.com/package/react-async-component) wraps the pattern in a `createAsyncComponent` call and provides server side rendering specific functionality.

## Conclusion

Code splitting is one of those features that allows you to push your application a notch further. You can load code when you need it to gain faster initial load times and improved user experience especially in a mobile context where bandwidth is limited.

To recap:

* **Code splitting** comes with extra effort as you have to decide what to split and where. Often, you find good split points within a router. Or you may notice that specific functionality is required only when a particular feature is used. Charting is a good example of this.
* In order to use dynamic `import` syntax, both Babel and ESLint require careful tweaks. Webpack supports the syntax ouf of the box.
* Dynamic `import` provides less functionality than `require.ensure`. While it's possible to handle errors with it, features like naming are available for `require.ensure` only.
* The techniques can be used within modern frameworks and libraries like React. You can wrap related logic to a specific component that handles the loading process in a user-friendly manner.

I will show you how to tidy up the build in the next chapter.

T> The *Searching with React* appendix contains a complete example of code splitting. It shows how to set up a static site index that's loaded when the user searches information.

T> [webpack-pwa](https://github.com/webpack/webpack-pwa) illustrates the idea on a larger scale and discusses different shell based approaches. We'll get back to this topic at the *Multiple Pages* chapter.
