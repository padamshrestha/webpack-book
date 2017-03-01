# Hot Module Replacement with React

Hot module replacement was one of the initial selling points of webpack and React. It relies on the [react-hot-loader](https://www.npmjs.com/package/react-hot-loader) package. At the time of writing, version 3 of *react-hot-loader* is in beta. It requires changes to three places: Babel configuration, webpack configuration, and application. I'll cover these next.

To get started, install the upcoming version of *react-hot-loader*:

```bash
npm install react-hot-loader@next --save-dev
```

## Setting Up Babel

To connect Babel with *react-hot-loader*, it needs to become aware of its plugin portion:

**.babelrc**

```json
{
leanpub-start-delete
  "plugins": ["syntax-dynamic-import"],
leanpub-end-delete
leanpub-start-insert
  "plugins": [
    "syntax-dynamic-import",
    "react-hot-loader/babel"
  ],
leanpub-end-insert
  ...
}
```

## Setting Up Webpack

On the webpack side, *react-hot-loader* requires an additional entry it uses to patch the running application. It is important the new entry runs first as otherwise, the setup will fail to work reliably:

**webpack.config.js**

```javascript
...

module.exports = function(env) {
  const pages = [
    ...
    parts.page({
      title: 'React demo',
      path: 'react',
      entry: {
leanpub-start-delete
        react: reactDemo,
leanpub-end-delete
leanpub-start-insert
        react: env === 'production' ? PATHS.reactDemo :
          ['react-hot-loader/patch', PATHS.reactDemo],
leanpub-end-insert
      },
      chunks: ['react', 'manifest', 'vendor'],
    }),
  ];
  ...
};
```

Patching is needed still as we have to make the application side aware of hot loading.

## Setting Up the Application

On React side, *react-hot-loader* relies on an `AppContainer` that deals with patching. We still have to implement the Hot Module Replacement interface as earlier. Set up an entry point for the demo as follows:

**app/react.js**

```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import Counter from './counter';
import { AppContainer } from 'react-hot-loader';

const app = document.createElement('div');
document.body.appendChild(app);

const render = App => {
  ReactDOM.render(
    <AppContainer><App /></AppContainer>,
    app
  );
};

render(Counter);

if (module.hot) {
  module.hot.accept('./counter', () => render(Counter));
}
```

In order to test the setup, a component is needed as well. In this case, it's going to be a little counter so you can see how the hot replacement mechanism maintains the state:

**app/counter.js**

```javascript
import React from 'react';

class Counter extends React.Component {
  constructor(props) {
    super(props);

    this.state = { amount: 0 };
  }
  render() {
    return (
      <div>
        <span className="fa fa-hand-spock-o fa-1g">
          Amount: {this.state.amount}
        </span>
        <button onClick={() => this.setState(addOne)}>
          Add one
        </button>
      </div>
    );
  }
}

const addOne = ({ amount }) => ({ amount: amount + 1 });

export default Counter;
```

If you run the application after these changes and modify the file above, it should pick up changes without a hard refresh while retaining the amount.

## Removing *react-hot-loader* Related Code from the Production Output

If you build the application (`npm run build`) and examine the output, you might spot references to `__REACT_HOT_LOADER__` there due to the Babel setup. It will use `react-hot-loader/babel` plugin regardless of the build target. To overcome this slight annoyance, we should configure Babel to apply the plugin only when we are developing.

Babel provides an [env option](https://babeljs.io/docs/usage/babelrc/#env-option) for this purpose. It respects both `NODE_ENV` and `BABEL_ENV` environment variables. If `BABEL_ENV` is set, it will receive precedence. To fix the issue, we can push the problematic Babel plugin behind a development specific `env` while controlling its behavior within webpack configuration by setting `BABEL_ENV`.

The webpack part should be adjusted like this:

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

Babel will now receive the target we pass to webpack allowing us to fix the behavior. Tweak Babel setup, so it matches the fields below. The key part is in pushing `react-hot-loader/patch` below `env`:

**.babelrc**

```json
{
leanpub-start-delete
  "plugins": [
    "syntax-dynamic-import",
    "react-hot-loader/babel"
  ],
leanpub-end-delete
leanpub-start-insert
  "plugins": ["syntax-dynamic-import"],
leanpub-end-insert
  ...
leanpub-start-insert
  "env": {
    "development": {
      "plugins": [
        "react-hot-loader/babel"
      ]
    }
  }
leanpub-end-insert
}
```

The development setup should work after this change still. If you examine the build output, you should notice it's missing references to `__REACT_HOT_LOADER__`.

Even after this change, the source might contain some references still due to a [bug in react-hot-loader](https://github.com/gaearon/react-hot-loader/issues/471) as it has been built so that it loses information that's valuable for a bundler.

It is possible to work around the issue by implementing a module chooser pattern as described in the *Setting Environment Variables* chapter. The idea is that `AppContainer` provided by *react-hot-loader* would be mocked with a dummy implementation during production usage.

T> The aforementioned `env` technique can be used to apply Babel presets and plugins per environment. You could enable additional checks and logging during development this way. See the *Loading JavaScript* chapter for more information.

## Configuring HMR with Redux

[Redux](http://redux.js.org/) is a popular state management library designed HMR in mind. To configure Redux reducers to support HMR, you have to implement the protocol as above:

```javascript
...

export default function configureStore(initialState) {
  const store = createStoreWithMiddleware(rootReducer, initialState);

  if(module.hot) {
    // Enable webpack hot module replacement for reducers
    module.hot.accept(
      '../reducers',
      () => store.replaceReducer(reducers)
    );
  }

  return store;
}
```

T> You can find [a full implementation of the idea online](https://github.com/survivejs-demos/redux-demo).

## Configuring Webpack to Work with JSX

Some people prefer to name their React components containing JSX using the `.jsx` suffix. Webpack can be configured to work with this convention. The benefit of doing this is that then your editor will be able to pick up the right syntax based on the file name alone. Another option is to configure the editor to use JSX syntax for `.js` files as it's a superset of JavaScript.

Webpack provides [resolve.extensions](https://webpack.js.org/guides/migrating/#resolve-extensions) field that can be used for configuring its extension lookup. If you want to allow imports like `import Button from './Button';` while naming the file as *Button.jsx*, set it up as follows:

```javascript
{
  resolve: {
    extensions: ['.js', '.jsx'],
  },
},
```

To resolve the problem at loader configuration, instead of matching against `/\.js$/`, we can expand it to include `.jsx` extension through `/\.(js|jsx)$/`. Another option would be to write `/\.jsx?$/`, but I find the explicit alternative more readable.

W> In webpack 1 you had to use `extensions: ['', '.js', '.jsx']` to match files without an extension too. This isn't needed in webpack 2.

## Get Started Fast with *create-react-app*

The fastest way to get started with webpack and React is to use [create-react-app](https://www.npmjs.com/package/create-react-app). It is a zero configuration approach that encapsulates a lot of best practices, and it is particularly useful if you want to get started with a little project fast with minimal setup.

One of the main attractions of *create-react-app* is a feature known as *ejecting* that allows you to extract a full-blown webpack setup out of it. There's a problem, though. After you eject, you cannot go back to the dependency-based model, and you will have to maintain the resulting setup yourself.

## Conclusion

*react-hot-loader* allows you to set up HMR with webpack. It was one of the initial selling points of both and is still a good technique. The setup takes some care, but after you have it running, it's cool.
