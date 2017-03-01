# Linting JavaScript

Linting is one of those techniques that can help you make fewer mistakes while coding JavaScript. You can spot issues before they become actual problems. Modern editors and IDEs offer strong support for popular tools allowing you to detect possible issues as you are developing.

Despite this, it is a good idea to set them up with webpack or at least in a separate task that gets run regularly. That allows you to cancel a production build that might not be up to your standards while enforcing teamwide standards.

## Brief Introduction to ESLint

The linter that started it all for JavaScript is Douglas Crockford's [JSLint](http://www.jslint.com/). JSLint is known to be opinionated like Douglas himself. The next step in evolution was [JSHint](http://jshint.com/), which took the opinionated edge out of JSLint and allowed for more customization. [ESLint](http://eslint.org/) is the newest tool in vogue, and it goes even further.

### ESLint Is Customizable

ESLint goes to the next level as it allows you to implement custom rules, parsers, and reporters. ESLint works with JSX syntax making it a good fit for React projects. You have to use a Babel specific parser with custom language features, although ESLint supports ES6 out of the box.

ESLint rules have been documented well, and you have full control over their severity. These features alone make it a powerful tool. Better yet, there is a significant number of rules and presets beyond the core as the community has built on top of it.

T> It is telling that a competing project, JSCS, [decided to merge its efforts with ESLint](http://eslint.org/blog/2016/04/welcoming-jscs-to-eslint). JSCS reached the end of its life with its 3.0.0 release, and the core team joined with ESLint.

### eslint-config-airbnb

[eslint-config-airbnb](https://www.npmjs.com/package/eslint-config-airbnb) is a good example of a popular preset. Often it is enough to find a preset you like, tweak it a little bit to your liking with some local rules or by deriving a preset of your own based on it, and then using that. This way you don't have to worry so much about all the available functionality.

T> [eslint-config-cleanjs](https://www.npmjs.com/package/eslint-config-cleanjs) is a good example of how you can use ESLint to restrict JavaScript to a purely functional subset.

## Linting Is about More than Catching Issues

Besides linting for issues, it can be useful to manage the code style on some level. Nothing is more annoying than having to work with source code that has mixed tabs and spaces. Stylistically consistent code reads better. Linting tools allow you to do this.

Establishing strong linting can be beneficial, especially in a context where you need to collaborate with others. Even when working alone you will benefit from linting as it can catch issues you might otherwise neglect. JavaScript as a language allows a lot of usages some of which, while valid, may not be the clearest to understand or may even be incorrect.

Linting does **not** replace proper testing, but it can complement testing approaches. It is one way to harden a codebase and make it a little harder to break. As the size of your project grows, and it becomes more challenging to manage, this becomes particularly important.

## Setting Up ESLint

![ESLint](images/eslint.png)

[ESLint](http://eslint.org/) is the most versatile linting solution for JavaScript. It builds on top of ideas presented by JSLint and JSHint. More importantly, it allows you to develop custom rules. As a result, a nice set of rules has been developed for React in the form of [eslint-plugin-react](https://www.npmjs.com/package/eslint-plugin-react).

T> Since *v1.4.0* ESLint supports [autofixing](http://eslint.org/blog/2015/09/eslint-v1.4.0-released/). It allows you to perform certain rule fixes automatically. To activate it, pass the flag `--fix` to the tool. It is also possible to use this feature with webpack, although you should be careful with it. [js-beautify](https://www.npmjs.com/package/js-beautify) can perform a similar operation.

### Connecting ESlint with *package.json*

To get started, install ESLint as a development dependency:

```bash
npm install eslint --save-dev
```

Next, we'll need to do some configuration so we can run ESLint smoothly through npm. I am using the `lint` namespace to signify it's a linting related task. I am also enabling caching to improve performance on subsequent runs. Add the following:

**package.json**

```json
...
"scripts": {
leanpub-start-insert
  "lint:js": "eslint . --cache",
leanpub-end-insert
  ...
},
...
```

Given ESLint expects configuration to work, we need to define some. It relies on rules that tell what to lint and how to react if the rule isn't obeyed. The severity of an individual rule is defined by a number as follows:

* 0 - The rule has been disabled.
* 1 - The rule will emit a warning.
* 2 - The rule will emit an error.

Some rules, such as `quotes`, accept an array instead allowing you to pass extra parameters to them. Refer to the [ESLint rules documentation](http://eslint.org/docs/rules/) for specifics.

Here's a starting point that will work with our project:

**.eslintrc.js**

```javascript
module.exports = {
  "env": {
    "browser": true,
    "commonjs": true,
    "es6": true,
    "node": true,
  },
  "extends": "eslint:recommended",
  "parserOptions": {
    "sourceType": "module",
  },
  "rules": {
    "comma-dangle": ["error", "always-multiline"],
    "indent": ["error", 2],
    "linebreak-style": ["error", "unix"],
    "quotes": ["error", "single"],
    "semi": ["error", "always"],
    "no-unused-vars": ["warn"],
    "no-console": 0,
  },
};
```

Also, we need to tell ESLint to skip linting the *build* directory by setting up ignore patterns. The *node_modules* directory is ignored by default.

**.eslintignore**

```bash
build/*
```

T> You can point ESLint to your Git ignores through `--ignore-path .gitignore`. It also accepts individual patterns, through `--ignore-pattern <pattern>`.

W> If you try a pattern like `build`, it will match files like *build.js* too! Remember to use a slash at the end to signify a directory.

If you invoke `npm run lint:js` now, it should execute without any warnings or errors. If you see either, this is a good time to try ESLint autofixing. You can run it like this: `npm run lint -- --fix`. Running an npm script this way allows you to pass extra parameters to it.

Another alternative would be to push it behind a *package.json* script. Autofix won't be able to repair each error, but it can fix a lot. And as time goes by and ESLint improves, it can perform more work.

Beyond vanilla JSON, ESLint supports other formats, such as JavaScript or YAML. I.e., *.eslintrc.yaml* would expect YAML. See the [documentation](http://eslint.org/docs/user-guide/configuring#configuration-file-formats) for further details.

T> When ESLint gives errors, npm will show a long `ELIFECYCLE error` error block of its own. It is possible to disable that using the `silent` flag like this: `npm run lint:js --silent` or a shortcut `npm run lint:js -s`.

### Connecting ESLint with Webpack

We can make webpack emit ESLint messages for us by using [eslint-loader](https://www.npmjs.com/package/eslint-loader). As the first step execute

```bash
npm install eslint-loader --save-dev
```

W> Note that *eslint-loader* will use a globally installed version of ESLint unless you have one included in the project itself. Make sure you have ESLint as a development dependency to avoid the strange behavior.

The loader needs some wiring to work. We'll discuss loaders in detail at the *Understanding Loaders* part, but the basic idea is fast to understand. A loader is connected to webpack through a rule that contains preconditions related to it and a reference to the loader itself.

In this case, we'll ensure that ESLint gets executed before anything else using the `enforce` field. It allows us to guarantee that linting happens before any other processing. The idea is discussed in detail at the *Loader Definitions* chapter.

To add linting to the project, adjust the configuration as follows:

**webpack.config.js**

```javascript
...


function developmentConfig() {
  const config = {
    devServer: {
      ...
    },
leanpub-start-insert
    module: {
      rules: [
        {
          test: /\.js$/,
          enforce: 'pre',

          loader: 'eslint-loader',
          options: {
            emitWarning: true,
          },
        },
      ],
    },
leanpub-end-insert
    plugins: [
      ...
    ],
  };

  ...
}

...
```

If you execute `npm start` now and break some linting rule while developing, you should see that in the terminal output.

W> Note that the webpack configuration lints only the application code we refer. If you want to lint webpack configuration itself, execute `npm run lint:js` separately.

T> It can be useful to attach the linting process to Git through a prepush hook. [pre-git](https://www.npmjs.com/package/pre-git) allows you to achieve this quickly. This way you can rebase your commits and fix possible problems before pushing.

### Configuring ESLint Further

Since webpack 2, the configuration schema of webpack has become stricter, and it doesn't allow arbitrary fields at configuration root level anymore. To overcome this issue and to access all functionality of the *eslint-loader*, you'll need to use `LoaderOptionsPlugin` as below:

```javascript
{
  plugins: [
    new webpack.LoaderOptionsPlugin({
      options: {
        eslint: {
          // Fail only on errors
          failOnWarning: false,
          failOnError: true,

          // Disable/enable autofix
          fix: false,

          // Output to Jenkins compatible XML
          outputReport: {
            filePath: 'checkstyle.xml',
            formatter: require('eslint/lib/formatters/checkstyle'),
          },
        },
      },
    }),
  ],
},
```

There are more options, and [eslint-loader](https://www.npmjs.com/package/eslint-loader) documentation covers those in detail.

## ESLint Tips

I've collected different ESLint tips below. The great thing about ESLint is that you can shape it to your purposes. The community around it is active, and you can find good integration in other tooling as well.

### Usability Tips

* Sometimes you might want to rely on some existing preset or set up custom configuration. That's where `--init` can come in handy. You can run it from `npm bin` and you'll end up with a call like `node_modules/.bin/eslint --init`
* ESLint supports custom formatters through `--format` parameter. [eslint-friendly-formatter](https://www.npmjs.com/package/eslint-friendly-formatter) is an example of a formatter that provides terminal-friendly output. This way you can jump conveniently straight to the warnings and errors from there.

### Performance Tips

* Especially on bigger projects it may be beneficial to run ESLint outside of webpack. That keeps code compilation fast while still giving the advantage of linting. Solutions like [lint-staged](https://www.npmjs.com/package/lint-staged) and [fastlint](https://www.npmjs.com/package/fastlint) can make this even more quickly.
* You can get more performance out of ESLint by running it through a daemon, such as [eslint_d](https://www.npmjs.com/package/eslint_d). Using it brings down the overhead, and it can bring down linting times considerably.

### Extension Tips

* ESLint supports ES6 features through configuration. You will have to specify the features to use through the [ecmaFeatures](http://eslint.org/docs/user-guide/configuring.html#specifying-language-options) property.
* There are useful plugins, such as [eslint-plugin-node](https://www.npmjs.com/package/eslint-plugin-node), [eslint-plugin-react](https://www.npmjs.com/package/eslint-plugin-react), [eslint-plugin-promise](https://www.npmjs.com/package/eslint-plugin-promise), [eslint-plugin-compat](https://www.npmjs.com/package/eslint-plugin-compat), and [eslint-plugin-import](https://www.npmjs.com/package/eslint-plugin-import), that you might want to consider integrating to your project.
* Most IDEs and editors have good linter integration so you can spot issues as you develop.
* To learn about ESLint customizations options and how to write an ESLint plugin, check out the *Customizing ESLint* appendix.

## Webpack and JSHint

No JSLint particular loader exists for webpack yet. Fortunately, there's one for JSHint. You could set it up on a legacy project quickly. The key is in configuring [jshint-loader](https://www.npmjs.com/package/jshint-loader).

JSHint will look into specific rules to apply from `.jshintrc`. You can also define custom settings within a `jshint` object at your webpack configuration. Exact configuration options have been covered by [the JSHint documentation](http://jshint.com/docs/) in detail.

## EditorConfig

[EditorConfig](http://editorconfig.org/) allows you to maintain a consistent coding style across different IDEs and editors. Some even come with built-in support. For others, you should install a separate plugin. Also, you'll need to set up a file like this:

**.editorconfig**

```yaml
root = true

# General settings for whole project
[*]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

# Format specific overrides
[*.md]
trim_trailing_whitespace = false

[app/**/*.js]
indent_style = space
indent_size = 2
```

T> [Prettier](https://www.npmjs.com/package/prettier) goes one step further and can format your code automatically according to your coding style. If you want to use Prettier with ESLint, you should use [eslint-config-prettier](https://www.npmjs.com/package/eslint-config-prettier) to disable ESLint rules that conflict with Prettier.

T> [Danger](https://github.com/danger/danger-js) operates on a higher level than the tools discussed. For example, it can check that the project change log was updated before a release is pushed to the public. You can also force pull requests of your project to comply specific standards.

## Conclusion

In this chapter, you learned how to lint your code using webpack in different ways. It is one of those techniques that yields benefits over the long term. You can fix possible problems before they become actual issues.

To recap:

* ESLint is the most versatile of the current options. You can expand it to fit your exact use case.
* ESLint can be run through webpack. It can terminate your build and even prevent it from getting deployed if your build does not pass the linting rules.
* EditorConfig complements ESLint by allowing you to define a project-level coding style. Editors integrate with EditorConfig making it easier to keep a project consistent regardless of the development platform.
* Prettier is an upcoming solution that can format your code automatically. Danger operates on repository level and can perform higher level tasks related to the development process.

Given the webpack configuration of our project is starting to get a little messy, and it won't get any easier should we extend it, it is a good time to discuss how to compose configuration and improve the situation further. We'll do that in the next chapter.
