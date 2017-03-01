# Loading JavaScript

Webpack processes ES6 module definitions by default and transforms them into code. It does **not** transform ES6 specific syntax apart, such as `const`. The resulting code can be problematic especially in the older browsers. It is also troublesome if you minify your code through UglifyJS as it doesn't support the ES6 syntax yet.

To get a better idea of the default transform, consider the example output below:

**build/app.js**

```javascript
webpackJsonp([1],{

/***/ 18:
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
/* harmony default export */ __webpack_exports__["a"] = function () {
  const element = document.createElement('div');

  element.className = 'fa fa-hand-spock-o fa-1g';
  element.innerHTML = 'Hello world';

  return element;
};

...
```

One way to work around this problem is to process the code through [Babel](https://babeljs.io/), a popular JavaScript compiler that supports ES6 features and more. It resembles ESLint in that it is built on top of presets and plugins. Presets are collections of plugins, and you can define your own as well.

T> Given sometimes extending existing presets might not be enough, [modify-babel-preset](https://www.npmjs.com/package/modify-babel-preset) allows you to go a step further and configure the base preset in a more flexible way.

## Using Babel with Webpack Configuration

Even though Babel can be used standalone, as you can see in the *Authoring Packages* chapter, you can hook it up with webpack as well. During development, we actually might skip processing.

Skipping processing is a good option especially if you don't rely on any custom language features and work using a modern browser. Processing through Babel becomes almost a necessity when you compile your code for production, though.

You can use Babel with webpack through [babel-loader](https://www.npmjs.com/package/babel-loader). It can pick up project level Babel configuration or you can configure it at the webpack loader itself.

Connecting Babel with a project allows you to process webpack configuration through it. To achieve this, name your webpack configuration using the *webpack.config.babel.js* convention. [interpret](https://www.npmjs.com/package/interpret) package enables this and it supports other compilers as well.

T> Given that [Node supports the ES6 specification well](http://node.green/) these days, you can use a lot of ES6 features without having to process configuration through Babel.

T> Babel isn't the only option although it is the most popular one. [Buble](https://buble.surge.sh) by Rich Harris is another compiler worth checking out. There's experimental [buble-loader](https://www.npmjs.com/package/buble-loader) that allows you to use it with webpack. Buble doesn't support ES6 modules, but that's not a problem as webpack provides that functionality.

### Setting Up *babel-loader*

The first step towards configuring Babel to work with webpack is to set up [babel-loader](https://www.npmjs.com/package/babel-loader). It will take our code and turn it into a format older browsers can understand. Install *babel-loader* and include its peer dependency *babel-core*:

```bash
npm install babel-loader babel-core --save-dev
```

As usual, let's define a part for Babel:

Here's the full loader configuration:

**webpack.parts.js**

```javascript
...

exports.loadJavaScript = function({ include, exclude }) {
  return {
    module: {
      rules: [
        {
          test: /\.js$/,
          include,
          exclude,

          loader: 'babel-loader',
          options: {
            // Enable caching for improved performance during
            // development.
            // It uses default OS directory by default. If you need
            // something more custom, pass a path to it.
            // I.e., { cacheDirectory: '<path>' }
            cacheDirectory: true,
          },
        },
      ],
    },
  };
};
```

Next, we need to connect this with the main configuration. If you are using a modern browser for development, you can consider processing only the production code through Babel. To play it safe, I'll use it for both production and development environments in this case.

Also, I'll constrain webpack to transform only our application code through Babel as I don't want it to process files from *node_modules* for example. It is a good practice with JavaScript files.

**webpack.config.js**

```javascript
...

const commonConfig = merge([
  {
  ...
leanpub-start-insert
  parts.loadJavaScript({ include: PATHS.app }),
leanpub-end-insert
]);

...
```

Even though we have Babel installed and set up, we are still missing one bit: Babel configuration. I prefer to handle it using a *.babelrc* dotfile as other tooling can pick it up as well.

W> There are times when caching Babel compilation can surprise you if your dependencies change in a way that *babel-loader* default caching mechanism doesn't notice. Override `cacheIdentifier` with a string that has been derived based on data that should invalidate the cache for better control. [Node crypto API](https://nodejs.org/api/crypto.html) and especially its MD5 related functions can come in handy.

W> If you try to import files **outside** of your configuration root directory and then process them through *babel-loader*, this will fail. It is [a known issue](https://github.com/babel/babel-loader/issues/313), and there are workarounds including maintaining *.babelrc* at a higher level in the project and resolving against Babel presets through `require.resolve` at webpack configuration.

### Setting Up *.babelrc*

At a minimum, you will need [babel-preset-es2015](https://www.npmjs.com/package/babel-preset-es2015). Install it:

```bash
npm install babel-preset-es2015 --save-dev
```

To make Babel aware of the preset, we need to write a *.babelrc*. Given webpack supports ES6 modules out of the box, we can tell Babel to skip processing them. Skipping this step would break webpack's HMR mechanism although the production build would still work.

**.babelrc**

```json
{
  "presets": [
    [
      "es2015",
      {
        "modules": false
      }
    ]
  ]
}
```

T> This is the Babel setup you would use if you want to support tree shaking at a package level. See the *Tree Shaking* chapter for more information.

If you execute `npm run build` now and examine *build/app.js*, you should see something a little different:

**build/app.js**

```javascript
webpackJsonp([1],{

/***/ 18:
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
/* harmony default export */ __webpack_exports__["a"] = function () {
  var element = document.createElement('div');

  element.className = 'fa fa-hand-spock-o fa-1g';
  element.innerHTML = 'Hello world';

  return element;
};

...
```

Note especially how the function was transformed. This code should work in older browsers now. It would also be possible to push it through UglifyJS without any errors due to parsing.

## Polyfilling Features

Given it's not always enough to transform ES6 code to an older format and expect it to work, polyfilling may be needed. The easiest way to solve this problem is to include [babel-polyfill](https://babeljs.io/docs/usage/polyfill/) to your project through an entry (`app: ['babel-polyfill', PATHS.app]`) or `import 'babel-polyfill'` from code to get it bundled.

It is important to note that *babel-polyfill* pollutes the global scope with objects like `Promise`. Given this can be problematic for library authors, there's [transform-runtime](https://babeljs.io/docs/plugins/transform-runtime/) option. It can be enabled as a Babel plugin, and it will avoid the problem of globals by rewriting the code in such way that they won't be needed.

## Babel Preset Based on Environment

Especially in bundle size sensitive environments *babel-polyfill* might not be the best option. If you know well which environment (browser versions, Node) you support, [babel-preset-env](https://www.npmjs.com/package/babel-preset-env) provides a more granular way to achieve the same result with a smaller size.

*babel-preset-env* is a forward-looking option. The advantage of using it is that then you don't have to worry about enabling specific language extensions specifically as it handles the work for you based on your target definition.

The problem is that if you want to minify your code, then you have to use tooling that supports ES6 as well. [babili](https://www.npmjs.com/package/babili) is a good option for that. The topic is discussed in detail in the **Minifying Build** chapter.

## Babel Tips

There are other possible [.babelrc options](https://babeljs.io/docs/usage/options/) beyond the ones covered here. Like ESLint, *.babelrc* supports [JSON5](https://www.npmjs.com/package/json5) as its configuration format meaning you can include comments in your source, use single quoted strings, and so on.

Sometimes you might want to use experimental features. Although you can find a lot of them within so-called stage presets, I recommend enabling them one by one and even organizing them to a preset of their own unless you are working on a throwaway project. If you expect your project to live a long time, it's better to document the features you are using well.

## Useful Babel Presets and Plugins

Perhaps the greatest thing about Babel is that it's possible to extend with presets and plugins. I've listed some interesting ones below:

* [babel-preset-es2015](https://www.npmjs.org/package/babel-preset-es2015) includes ES2015 features.
* [babel-preset-es2016](https://www.npmjs.org/package/babel-preset-es2016) includes **only** ES2016 features. Remember to include the previous preset as well if you want both!
* [babel-plugin-import](https://www.npmjs.com/package/babel-plugin-import) rewrites module imports so that you can use a form such as `import { Button } from 'antd';` instead of pointing to the module through an exact path.
* [babel-plugin-import-asserts](https://www.npmjs.com/package/babel-plugin-import-asserts) asserts that your imports have been defined.
* [babel-plugin-log-deprecated](https://www.npmjs.com/package/babel-plugin-log-deprecated) adds `console.warn` to functions that have `@deprecate` annotation in their comment.
* [babel-plugin-annotate-console-log](https://www.npmjs.com/package/babel-plugin-annotate-console-log) annotates `console.log` calls with information about invocation context so it's easier to see where they logged.
* [babel-plugin-webpack-loaders](https://www.npmjs.com/package/babel-plugin-webpack-loaders) allows you to use certain webpack loaders through Babel.
* [babel-plugin-syntax-trailing-function-commas](https://www.npmjs.com/package/babel-plugin-syntax-trailing-function-commas) adds trailing comma support for functions.
* [babel-react-optimize](https://github.com/thejameskyle/babel-react-optimize) implements a variety of React specific optimizations you may want to experiment with.
* [babel-plugin-transform-react-remove-prop-types](https://www.npmjs.com/package/babel-plugin-transform-react-remove-prop-types) allows you to remove `propType` related code from your production build. It also allows component authors to generate code that's wrapped so that setting environment at `DefinePlugin` can kick in as discussed in the book.

T> It is possible to connect Babel with Node through [babel-register](https://www.npmjs.com/package/babel-register) or [babel-cli](https://www.npmjs.com/package/babel-cli). These packages can be handy if you want to execute your code through Babel without using webpack.

## Enabling Presets and Plugins per Environment

Babel allows you to control which presets and plugins are used per environment through its [env option](https://babeljs.io/docs/usage/babelrc/#env-option). You can manage Babel's behavior per build target this way.

`env` checks both `NODE_ENV` and `BABEL_ENV` and functionality to your build based on that. If `BABEL_ENV` is set, it will override any possible `NODE_ENV`. Consider the example below:

```json
{
  ...
  "env": {
    "development": {
      "plugins": [
        "react-hot-loader/babel"
      ]
    }
  }
}
```

Note that any shared presets and plugins are available to all targets still. `env` allows you to specialize your Babel configuration further.

It is possible to pass the webpack environment to Babel with a tweak like this:

**webpack.config.js**

```javascript
...

module.exports = function(env) {
leanpub-start-insert
  process.env.BABEL_ENV = env;
leanpub-end-insert

  ...
};
```

T> The way `env` works is subtle. Consider logging `env` and make sure it matches your Babel configuration or otherwise the functionality you expect might not get applied to your build.

T> The technique is used in the *Server Side Rendering* chapter to enable the Babel portion of *react-hot-loader* for development target only.

## Setting Up TypeScript

Microsoft's [TypeScript](http://www.typescriptlang.org/) is a compiled language that follows a similar setup as Babel. The neat thing is that in addition to JavaScript, it can emit type definitions. A good editor can pick those up and provide enhanced editing experience. Stronger typing is useful for development as it becomes easier to state your type contracts.

Compared to Facebook's type checker Flow, TypeScript is a more established option. As a result, you will find more premade type definitions for it, and overall, the quality of support should be better.

You can use TypeScript with webpack using the following loaders:

* [ts-loader](https://www.npmjs.com/package/ts-loader)
* [awesome-typescript-loader](https://www.npmjs.com/package/awesome-typescript-loader)
* [light-ts-loader](https://www.npmjs.com/package/light-ts-loader)

T> There's a [TypeScript parser for ESLint](https://www.npmjs.com/package/typescript-eslint-parser). It's also possible to lint it through [tslint](https://www.npmjs.com/package/tslint).

## Setting Up Flow

[Flow](http://flowtype.org/) performs static analysis based on your code and its type annotations. You have to install it as a separate tool and then run it against your code. There's [flow-status-webpack-plugin](https://www.npmjs.com/package/flow-status-webpack-plugin) that allows you to run it through webpack during development.

If you use React, the React specific Babel preset does most of the work through [babel-plugin-syntax-flow](https://www.npmjs.com/package/babel-plugin-syntax-flow). It can strip Flow annotations and convert your code into a format that is possible to transpile further.

There's also [babel-plugin-typecheck](https://www.npmjs.com/package/babel-plugin-typecheck) that allows you to perform runtime checks based on your Flow annotations. [flow-runtime](https://codemix.github.io/flow-runtime/) goes a notch further and provides more functionality. These approaches complement Flow static checker and allow you to catch even more issues.

T> [flow-coverage-report](https://www.npmjs.com/package/flow-coverage-report) shows how much of your code is covered by Flow type annotations.

## Conclusion

Babel has become an indispensable tool for developers given it bridges the standard with older browsers. Even if you targeted modern browsers, transforming through Babel may be useful.

To recap:

* Babel gives you control over what browsers to support. It can compile ES6 features to a form the older browser understand.
* Babel allows you to use experimental language features. You can find numerous plugins that improve development experience and the production build through optimizations.
* Babel functionality can be enabled per development target. This way you can be sure you are using the correct plugins at the right place.
* Besides Babel, webpack supports other solutions like TypeScript of Flow. Flow can complement Babel while TypeScript represents an entire language compiling to JavaScript.
