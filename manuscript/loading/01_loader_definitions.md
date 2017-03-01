# Loader Definitions

Webpack provides multiple ways to set up module loaders. Webpack 2 simplified the situation by introducing the `use` field. The legacy options (`loader` and `loaders`) still work, though. I'll discuss all the options for completeness, as you may see them in existing configurations.

It can be a good idea to prefer absolute paths here as they allow you to move configuration without breaking assumptions. The other option is to set `context` field as this gives a similar effect and affects the way entry points and loaders are resolved. It won't have an impact on the output, though, and you still need to use an absolute path or `/` there.

Assuming you set an `include` or `exclude` rule, packages loaded from *node_modules* will still work as the assumption is that they have been compiled in such way that they work out of the box. Sometimes you may come upon a poorly packaged one, but often you can work around these by tweaking your loader configuration or setting up a `resolve.alias` against an asset that is included in the offending package.

T> The *Consuming Packages* chapter discusses the aliasing idea in further detail.

T> `include`/`exclude` is particularly useful with *node_modules* as webpack will process and traverse the installed packages by default when you import JavaScript files to your project. Therefore you need to configure it to avoid that behavior. Other file types don't suffer from this issue.

## Anatomy of a Loader

Webpack supports a large variety of formats through *loaders*. Also, it supports a couple of JavaScript module formats out of the box. The idea is the same. You always set up a loader, or loaders, and connect those with your directory structure.

Consider the example below where webpack is set to process JavaScript through Babel:

**webpack.config.js**

```javascript
...

module.exports = {
  ...
  module: {
    rules: [
      {
        // Conditions

        // Match files against RegExp. This accepts
        // a function too.
        test: /\.js$/,

        // Restrict matching to a directory. This
        // also accepts an array of paths.
        //
        // Although optional, I prefer to set this for
        // JavaScript source as it helps with
        // performance and keeps the configuration cleaner.
        //
        // This accepts an array or a function too. The
        // same applies to `exclude` as well.
        include: path.join(__dirname, 'app'),

        /*
        exclude(path) {
          // You can perform more complicated checks
          // through functions if you want.
          return path.match(/node_modules/);
        },
        */

        // Actions

        // Apply loaders the matched files. These need to
        // be installed separately. In this case our
        // project would need *babel-loader*. This
        // accepts an array of loaders as well and
        // more forms are possible as discussed below.
        use: 'babel-loader',
      },
    ],
  },
};
```

T> If you are not sure how a particular RegExp matches, consider using an online tool, such as [regex101](https://regex101.com/) or [RegExr](http://regexr.com/).

T> Babel is discussed in detail in the *Loading JavaScript* chapter. We'll attach it to the book project there.

## Loader Evaluation Order

It is good to keep in mind that webpack's `loaders` are always evaluated from right to left and from bottom to top (separate definitions). The right-to-left rule is easier to remember when you think about as functions. You can read definition `use: ['style-loader', 'css-loader']` as `style(css(input))` based on this rule.

To see the rule in action, consider the example below:

```javascript
{
  test: /\.css$/,
  use: ['style-loader', 'css-loader'],
}
```

Based on the right to left rule, the example can be split up while keeping it equivalent:

```javascript
{
  test: /\.css$/,
  use: ['style-loader'],
},
{
  test: /\.css$/,
  use: ['css-loader'],
},
```

### Enforcing Order

Even though it would be possible to develop an arbitrary configuration using the rule above, it can be convenient to be able to force certain rules to be applied before or after regular ones. The `enforce` field can come in handy here. It can be set to either `pre` or `post` to push processing either before or after other loaders.

We used the idea earlier in the *Linting JavaScript* chapter. Linting is a good example as the build should fail before it does anything else. Using `enforce: 'post'` is rarer and it would imply you want to perform a check against the built source. Performing analysis against the built source is one potential example.

The basic syntax goes like this:

```javascript
{
  // Conditions
  test: /\.js$/,
  enforce: 'pre',

  // Actions
  loader: 'eslint-loader',
},
```

It would be possible to write the same configuration without `enforce` if you chained the declaration with other loaders related to the `test` carefully. Using `enforce` removes the necessity for that allows you to split loader execution into separate stages that are easier to compose.

## Passing Parameters to a Loader

There's a query format that allows passing parameters to loaders:

```javascript
{
  // Conditions
  test: /\.js$/,
  include: PATHS.app,

  // Actions
  use: 'babel-loader?cacheDirectory,presets[]=es2015',
},
```

It is good to note that this style of configuration works in entries and source imports too as webpack will pick it up. The format may come in handy in certain individual cases, but often you are better off using more readable alternatives.

It is preferable to use the combination of `loader` and `options` fields either like this:

```javascript
{
  // Conditions
  test: /\.js$/,
  include: PATHS.app,

  // Actions
  loader: 'babel-loader',
  options: {
    cacheDirectory: true,
    presets: ['react', 'es2015'],
  },
},
```

Or you can also go through `use` like this:

```javascript
{
  // Conditions
  test: /\.js$/,
  include: PATHS.app,

  // Actions
  use: {
    loader: 'babel-loader',
    options: {
      cacheDirectory: true,
      presets: ['react', 'es2015'],
    },
  },
},
```

If we wanted to use more than one loader, we could pass an array to `use` and expand from there:

```javascript
{
  test: /\.js$/,
  include: PATHS.app,

  use: [
    {
      loader: 'babel-loader',
      options: {
        cacheDirectory: true,
        presets: ['react', 'es2015'],
      },
    },
    // Add more loaders here
  ],
},
```

## Branching at `use` Using a Function

In the book setup, we compose configuration on a higher level. Another option to achieve similar results would be to branch at `use` as webpack's loader definitions accept functions that allow you to branch depending on the environment. Consider the example below:

```javascript
{
  test: /\.css$/,

  // resource refers to the resource path matched.
  // resourceQuery contains possible query passed to it (?sourceMap)
  // issuer tells about match context path
  use: ({ resource, resourceQuery, issuer }) => {
    // You have to return either something falsy,
    // string (i.e., 'style-loader'), or an object
    // from here.
    //
    // Returning an array will fail! To get around that,
    // it is possible to nest rules.
    if (env === 'development') {
      return {
        // Trigger css-loader first
        loader: 'css-loader',
        rules: [
          // And style-loader after it
          'style-loader',
        ],
      };
    }

    // Do something else in production environment now
    ...
  },
},
```

Carefully applied, this technique can be useful and allow different ways of composition.

## Inline Definitions

Even though configuration level loader definitions are preferable, it is possible to write loader definitions inline like this:

```javascript
// Process foo.png through url-loader and other
// possible matches.
import 'url-loader!./foo.png';

// Override possible higher level match completely
import '!!url-loader!./bar.png';
```

The problem with this approach is that it couples your source with webpack. But it is a good form to know still. Since configuration entries go through the same mechanism, the same forms work there as well:

```javascript
{
  entry: {
    app: 'babel-loader!./app',
  },
},
```

## Alternate Ways to Match Files

`test` combined with `include` or `exclude` to constrain the match is the most common way to match files. These accept the data types as listed below:

* `test` - Match against a RegExp, string, function, an object, or an array of conditions like these.
* `include` - The same.
* `exclude` - The same, except the output is the inverse of `include`.

There are a couple of boolean based fields that can be used to constrain the result further:

* `not` - Do **not** match against a condition like above.
* `and` - Match against an array of conditions. All must match.
* `or` - Match against an array while any must match.

Webpack can also match based on the resource path and related information:

* `resource: /inline/` - Match against a resource path including the query. Example matches: `/path/foo.inline.js`, `/path/bar.png?inline`.
* `issuer: /bar.js/` - Match against a resource requested from the match. Example: `/path/foo.png` would match if it was requested from `/path/bar.js`.
* `resourcePath: /inline/` - Match against a resource path without its query. Example match: `/path/foo.inline.png`.
* `resourceQuery: /inline/` - Match against a resource based on its query. Example match: `/path/foo.png?inline`.

The fields above can be combined to apply different loaders based on the context with the `oneOf` field. To handle images differently based on their query, we could apply a rule like this:

```javascript
{
  test: /\.css$/,
  oneOf: [
    {
      resourceQuery: /inline/,
      use: 'url-loader',
    },
    {
      resourceQuery: /external/,
      use: 'file-loader',
    },
  ],
}
```

If you wanted to embed the context information to the filename, the rule could use `resourcePath` over `resourceQuery`.

## `LoaderOptionsPlugin`

Given webpack 2 forbids arbitrary root level configuration, you have to use `LoaderOptionsPlugin` to manage it. The plugin exists for legacy compatibility and may disappear in a future release. Consider the example below:

```javascript
{
  plugins: [
    new webpack.LoaderOptionsPlugin({
      sassLoader: {
        includePaths: [
          path.join(__dirname, 'style'),
        ],
      },
    }),
  ],
},
```

## Conclusion

Webpack provides multiple ways to set up loaders, but sticking with `use` is enough in webpack 2. You should be careful, especially with loader ordering, as this is a common source of problems.

To recap:

* **Loaders** allow you determine what should happen when webpack's module resolution mechanism encounters a file.
* A loader definition consists of **conditions** based on which to match and **actions** that should be performed when a match happens.
* Webpack 2 introduced the `use` field. It combines the ideas of old `loader` and `loaders` fields into a single construct.
* Webpack 2 provides multiple ways to match and alter loader behavior. You can, for example, match based on a **resource query** after a loader has been matched and route the loader to specific actions.
* `LoaderOptionsPlugin` exists for legacy purposes and allows you to get around the strict configuration schema of webpack 2 to work with older plugins and loaders.

In the next chapter, I will show you how to load images using webpack.
