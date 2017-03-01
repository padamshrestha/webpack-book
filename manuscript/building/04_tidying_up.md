# Tidying Up

The current setup doesn't clean the *build* directory between builds. As a result, it will keep on accumulating files as our project changes. Given this can get annoying, we should clean it up in between.

Another nice touch would be to include information about the build itself to the generated bundles as a small comment at the top of each file including version information at least.

## Cleaning the Build Directory

This issue can be resolved either by using a webpack plugin or solving it outside of it. You could trigger `rm -rf ./build && webpack` or `rimraf ./build && webpack` in an npm script to keep it cross-platform. A task runner could work for this purpose as well.

### Setting Up `CleanWebpackPlugin`

Install the [clean-webpack-plugin](https://www.npmjs.com/package/clean-webpack-plugin) first:

```bash
npm install clean-webpack-plugin --save-dev
```

Next, we need to define a little function to wrap the basic idea. We could use the plugin directly, but this feels like something that could be useful across projects, so it makes sense to push it to our library:

**webpack.parts.js**

```javascript
...
leanpub-start-insert
const CleanWebpackPlugin = require('clean-webpack-plugin');
leanpub-end-insert

...

leanpub-start-insert
exports.clean = function(path) {
  return {
    plugins: [
      new CleanWebpackPlugin([path]),
    ],
  };
};
leanpub-end-insert
```

Connect it with our project like this:

**webpack.config.js**

```javascript
...

const productionConfig = merge([
leanpub-start-insert
  parts.clean(PATHS.build),
leanpub-end-insert
  ...
]);

...
```

After this change, our `build` directory should remain nice and tidy while building. You can verify this by building the project and making sure no old files remained in the output directory.

## Attaching a Revision to the Build

It can be useful to attach information about the current build revision to the build files themselves. The desired result can be achieved using [webpack.BannerPlugin](https://webpack.js.org/plugins/banner-plugin/). It can be used in combination with [git-revision-webpack-plugin](https://www.npmjs.com/package/git-revision-webpack-plugin) to generate a small comment at the beginning of the generated files.

### Setting Up `BannerPlugin` and `GitRevisionPlugin`

To get started, install the revision plugin:

```bash
npm install git-revision-webpack-plugin --save-dev
```

Then define a part to wrap the idea:

**webpack.parts.js**

```javascript
...
leanpub-start-insert
const GitRevisionPlugin = require('git-revision-webpack-plugin');
leanpub-end-insert

...

leanpub-start-insert
exports.attachRevision = function() {
  return {
    plugins: [
      new webpack.BannerPlugin({
        banner: new GitRevisionPlugin().version(),
      }),
    ],
  };
};
leanpub-end-insert
```

And connect it to the main configuration:

**webpack.config.js**

```javascript
...

const productionConfig = merge([
  parts.clean(PATHS.build),
leanpub-start-insert
  parts.attachRevision(),
leanpub-end-insert
  ...
]);

...
```

If you build the project (`npm run build`), you should notice the built files contain comments like `/*! 0b5bb05 */` or `/*! v1.7.0-9-g5f82fe8 */` in the beginning.

The output can be customized further by adjusting the banner. You can also pass revision information to the application using `webpack.DefinePlugin`. This technique is discussed in detail at the *Setting Environments Chapter*.

W> The code expects you run it within a Git repository! Otherwise, you will get a `fatal: Not a git repository (or any of the parent directories): .git` error. If you are not using Git, you can replace the banner with some other data.

## Copying Files

Copying files is another common operation you can handle with webpack. [copy-webpack-plugin](https://www.npmjs.com/package/copy-webpack-plugin) can be handy if you need to bring external files to your build without having webpack pointing at them directly.

[cpy-cli](https://www.npmjs.com/package/cpy-cli) is a good option if you want to copy outside of webpack in a cross-platform way. Plugins should be cross-platforms by definition.

## Conclusion

Often, you work with webpack like this: First, you identify a problem and then find a plugin to tackle it. It is entirely acceptable to solve these types of issues outside of webpack, but webpack can often handle them as well.

To recap:

* You can find many small plugins that work as tasks and push webpack closer to a task runner.
* These tasks include cleaning the build and deployment. The *Deploying Applications* chapter discusses the latter topic in detail.
* It can be a good idea to include small comments to the production build to tell what version has been deployed. This way you can debug potential issues faster.
* Secondary tasks like these can be performed outside of webpack. If you are using a multi-page setup as discussed in the *Multiple Pages* chapter, this will become a necessity.
