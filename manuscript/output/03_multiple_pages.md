# Multiple Pages

Even though webpack is often used for bundling single page applications, it is possible to use it with multiple separate pages as well. The idea is similar to the way we generated multiple output files in the *Targets* chapter. This time, however, we have to generate separate pages. That's achievable through `HtmlWebpackPlugin` and a bit of configuration.

## Possible Approaches

When generating multiple pages with webpack, you have a couple of considerations:

* Go through the *multi-compiler mode* and return an array of configurations. The approach would work as long as the pages are separate and there is a minimal need for sharing code across them. The benefit of this approach is that you can process it through [parallel-webpack](https://www.npmjs.com/package/parallel-webpack) to improve build performance.
* Set up a single configuration and extract the commonalities. The way you do this can differ depending on how you chunk it up.
* If you follow the idea of [Progressive Web Applications](https://developers.google.com/web/progressive-web-apps/) (PWA), you can end up with either an **app shell** or a **page shell** and load portions of the application as it is used.

In practice, you may find more dimensions. For example, you have to generate i18n variants for pages. These ideas grow on top of the basic approaches.

## Generating Multiple Pages

To generate multiple separate pages, they should be initialized somehow. We should also be able to return a configuration for each page, so webpack will pick them up and process them through the multi-compiler mode.

### Modeling a Configuration Part

To initialize a page, it should receive page title, output path, and an optional template at least. Each page should receive optional output path, and a template for customization. The idea can be modeled as a configuration part like this:

**webpack.parts.js**

```javascript
...
leanpub-start-insert
const HtmlWebpackPlugin = require('html-webpack-plugin');
leanpub-end-insert

...

leanpub-start-insert
exports.page = function({
  path = '',
  template = require.resolve(
    'html-webpack-plugin/default_index.ejs'
  ),
  title,
} = {}) {
  return {
    plugins: [
      new HtmlWebpackPlugin({
        filename: `${path && path + '/'}index.html`,
        template,
        title,
      }),
    ],
  };
};
leanpub-end-insert
```

### Integrating to Main Configuration

To incorporate the idea to the configuration, the way it is composed has to change. Also, a page definition is required. To get started, let's reuse the same JavaScript logic for each page for now:

**webpack.config.js**

```javascript
const path = require('path');
leanpub-start-delete
const HtmlWebpackPlugin = require('html-webpack-plugin');
leanpub-end-delete
...


const commonConfig = merge([
  {
    ...
leanpub-start-delete
    plugins: [
      new HtmlWebpackPlugin({
        title: 'Webpack demo',
      }),
    ],
leanpub-end-delete
  },
]);

...

module.exports = function(env) {
leanpub-start-delete
  if (env === 'production') {
    return merge(commonConfig, productionConfig);
  }

  return merge(commonConfig, developmentConfig);
leanpub-end-delete
leanpub-start-insert
  const pages = [
    parts.page({ title: 'Webpack demo' }),
    parts.page({ title: 'Another demo', path: 'another' }),
  ];
  const config = env === 'production' ?
    productionConfig :
    developmentConfig;

  return pages.map(page => merge(commonConfig, config, page));
leanpub-end-insert
};
```

After this change you should have two pages in the application: `/` and `/another`. It should be possible to navigate to both while seeing the same output.

### Injecting Different Script per Page

The question is, how to inject a different script per each page. In the current configuration, the same `entry` is shared by both. To solve the problem, we should move `entry` configuration to lower level and manage it per page. To have a script to test with, set up another entry point like this:

**app/another.js**

```javascript
import './main.css';
import component from './component';

let demoComponent = component('Another');

document.body.appendChild(demoComponent);
```

The file could go to a directory of its own. Here we reuse the existing code to get something to show up. Webpack configuration has to point to this file:

**webpack.config.js**

```javascript
...

const commonConfig = merge([
  {
leanpub-start-delete
    entry: {
      app: PATHS.app,
    },
leanpub-end-delete
    ...
  },
  ...
]);

...

module.exports = function(env) {
leanpub-start-delete
  const pages = [
    parts.page({ title: 'Webpack demo' }),
    parts.page({ title: 'Another demo', path: 'another' }),
  ];
leanpub-end-delete
leanpub-start-insert
  const pages = [
    parts.page({
      title: 'Webpack demo',
      entry: {
        app: PATHS.app,
      },
    }),
    parts.page({
      title: 'Another demo',
      path: 'another',
      entry: {
        another: path.join(PATHS.app, 'another.js'),
      },
    }),
  ];
leanpub-end-insert
  const config = env === 'production' ?
    productionConfig :
    developmentConfig;

  return pages.map(page => merge(commonConfig, config, page));
};
```

The tweak also requires a change at the related part so that `entry` gets included in the configuration:

**webpack.parts.js**

```javascript
...

exports.page = function({
  ...
leanpub-start-insert
  entry,
leanpub-end-insert
} = {}) {
  return {
leanpub-start-insert
    entry,
leanpub-end-insert
    ...
  };
};
```

After these changes `/another` should show something familiar:

![Another page shows up](images/another.png)

### Pros and Cons

If you build the application (`npm run build`), you should find *another/index.html*. Based on the generated code, we can make the following observations:

* It is easy to add more pages to the setup.
* The generated assets are directly below the build root. The pages are an exception as those are handled by `HtmlWebpackPlugin`, but they still point to the assets below the root. It would be possible to add more abstraction in the form of *webpack.page.js* and manage the paths by exposing a function that accepts page configuration.
* Records should be written separately per each page in files of their own. Currently, the configuration that writes the last, wins. The above solution would allow solving this.
* Processes like linting and cleaning run twice currently. The *Targets* chapter discussed potential solutions to that problem.

The approach can be pushed to another direction by dropping the multi-compiler mode. Even though it's slower to process a build like this, it enables code sharing, and the implementation of shells. The first step towards a shell setup is to rework the configuration so that it picks up the code shared between the pages.

## Generating Multiple Pages While Sharing Code

Our current configuration shares code by coincidence already due to the usage patterns. Only a small part of the code differs, and as a result only the page manifests, and the bundles mapping to their entries differ.

In a more complicated application, you might want to apply techniques covered in the *Splitting Bundles* chapter across the pages. Dropping the multi-compiler mode can become useful then.

### Adjusting Configuration

To reach a code sharing setup, a minor adjustment is needed. Most of the code can remain the same. The way we expose it to webpack has to change so that it receives a single configuration object. As `HtmlWebpackPlugin` picks up all chunks by default, we have to adjust it to pick up only the chunks that are related to each page:

**webpack.config.js**

```javascript
...

module.exports = function(env) {
  const pages = [
    parts.page({
      title: 'Webpack demo',
      entry: {
        app: PATHS.app,
      },
leanpub-start-insert
      chunks: ['app', 'manifest', 'vendor'],
leanpub-end-insert
    }),
    parts.page({
      title: 'Another demo',
      path: 'another',
      entry: {
        another: path.join(PATHS.app, 'another.js'),
      },
leanpub-start-insert
      chunks: ['another', 'manifest', 'vendor'],
leanpub-end-insert
    }),
  ];
  const config = env === 'production' ?
    productionConfig :
    developmentConfig;

leanpub-start-delete
  return pages.map(page => merge(commonConfig, config, page));
leanpub-end-delete
leanpub-start-insert
  return merge([commonConfig, config].concat(pages));
leanpub-end-insert
};
```

The page-specific configuration requires a small tweak as well:

**webpack.parts.js**

```javascript
...

exports.page = function({
  ...
leanpub-start-insert
  chunks,
leanpub-end-insert
} = {}) {
  return {
    entry,
    plugins: [
      new HtmlWebpackPlugin({
leanpub-start-insert
        chunks,
leanpub-end-insert
        ...
      }),
    ],
  };
};
```

If you generate a build (`npm run build`), you should notice that something is a little different compared to the first multiple page build we did. Instead of two manifest files, you can find only one. If you examine it, you will notice it contains references to all files that were generated.

Studying the entry specific files in detail reveals more. You can see that they point to different parts of the manifest. The manifest will run different code depending on the entry. Multiple separate manifests are not needed.

### Pros and Cons

Compared to the earlier approach, something was gained, but also lost:

* Given the configuration isn't in the multi-compiler form anymore, processing it will be slower.
* Plugins such as `CleanWebpackPlugin` will work without additional consideration now.
* Instead of multiple manifests, only one remains. The result is not a problem, though, as the entries use it differently based on their setup.
* `CommonsChunkPlugin` related setup required careful thought to avoid problems with styling. The earlier approach avoided this issue through isolation.

## Progressive Web Applications

The idea can be pushed further if you combine it with code splitting and smart routing and this is the idea of PWAs. Webpack is well suited to the approach as illustrated by the [webpack-pwa](https://github.com/webpack/webpack-pwa) example.

The PWA example implements client-side routing. When you go from a page to another, it loads the functionality needed using a dynamic `import`. It implements two approaches - app shell and page shell.

App shell is loaded initially, and it manages the whole application including its routing. Compared to app shells, page shells are more granular, and more are loaded as you use the application. The total size of the application is larger in this case, but conversely, you can load initial content faster.

The approach combines well with plugins like [offline-plugin](https://www.npmjs.com/package/offline-plugin) and [sw-precache-webpack-plugin](https://www.npmjs.com/package/sw-precache-webpack-plugin). This way you can benefit from [Service Workers](https://developer.mozilla.org/en/docs/Web/API/Service_Worker_API) and improve the offline experience.

## Conclusion

Webpack allows you to manage multiple page setups with some thought. Especially the PWA approach is interesting as it allows loading the application in a progressive manner based on usage. It is more flexible than a more traditional single page application approach, and webpack's functionality enables it well.

To recap:

* Webpack can be used to generate separate pages either through its multi-compiler mode or by including all the page configuration into one.
* The multi-compiler configuration can run in parallel using external solutions, but it is harder to apply techniques such as bundle splitting against it.
* A multi-page setup can lead to a **Progressive Web Application**. In this case you'll use various webpack techniques to come up with an application that is fast to load and that fetches functionality as required. Both two flavors of this technique have their own merits.

I will discuss the idea of Server Side Rendering in the next chapter.
