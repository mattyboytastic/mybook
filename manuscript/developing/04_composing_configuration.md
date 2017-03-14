# Composing Configuration

Even though not a lot has been done with webpack yet, the amount of configuration is starting to feel substantial. Also, you have to be careful about the way you compose it as you have separate production and development targets in the project now. The situation can only get worse as you want to add more functionality to the project.

Using a single monolithic configuration file impacts comprehension and removes any potential for reusablity. As the needs of your project grow, you have to figure out the means to manage webpack configuration more effectively.

## Possible Ways to Manage Configuration

You can manage webpack configuration in the following ways:

* Maintain configuration in multiple files for each environment and point webpack to each through the `--config` parameter, sharing configuration through module imports. You can see this approach in action at [webpack/react-starter](https://github.com/webpack/react-starter).
* Push configuration to a library, which you then consume. Example: [HenrikJoreteg/hjs-webpack](https://www.npmjs.com/package/hjs-webpack).
* Maintain all configuration within a single file and branch there and by relying on the `--env` parameter.

These approaches can be combined to create a higher level configuration that's then composed of smaller parts. Those parts could then be added to a library which you then use through npm making it possible to consume the same configuration across multiple projects.

This approach is used to discuss different techniques. *webpack.config.js* maintains higher level configuration while *webpack.parts.js* contains the building blocks.

## Composing Configuration by Merging

If the configuration file is broken into separate pieces, they have to be combined together again somehow. Normally this means merging objects and arrays. To eliminate the problem of dealing with `Object.assign` and `Array.concat`, [webpack-merge](https://www.npmjs.org/package/webpack-merge) was developed.

*webpack-merge* does two things: it concatenates arrays and merges objects instead of overriding them. Even though a basic idea, this allows you to compose configuration and gives a degree of abstraction.

The example below shows the behavior in detail:

```bash
> merge = require('webpack-merge')
...
> merge(
... { a: [1], b: 5, c: 20 },
... { a: [2], b: 10, d: 421 }
... )
{ a: [ 1, 2 ], b: 10, c: 20, d: 421 }
```

*webpack-merge* provides even more control through strategies that enable you to control its behavior per field. Strategies allow you to force it to append, prepend, or replace content. Even though *webpack-merge* was designed for this book, it has proven to be an invaluable tool beyond it. You can consider it as a learning tool and pick it up in your work if you find it handy.

T> [webpack-chain](https://www.npmjs.com/package/webpack-chain) provides a fluent API for configuring webpack allowing you to avoid configuration shape-related problems while enabling composition.

{pagebreak}

## Setting Up *webpack-merge*

To get started, add *webpack-merge* to the project:

```bash
npm install webpack-merge --save-dev
```

Next, you need to refactor *webpack.config.js* into parts you can consume from there and then rewrite the file to use the parts. Here are the parts with small function-based interfaces extracted from the existing code:

**webpack.parts.js**

```javascript
exports.devServer = ({ host, port } = {}) => ({
  devServer: {
    historyApiFallback: true,
    stats: 'errors-only',
    host, // Defaults to `localhost`
    port, // Defaults to 8080
    overlay: {
      errors: true,
      warnings: true,
    },
  },
});

exports.lintJavaScript = ({ include, exclude, options }) => ({
  module: {
    rules: [
      {
        test: /\.js$/,
        include,
        exclude,
        enforce: 'pre',

        loader: 'eslint-loader',
        options,
      },
    ],
  },
});
```

T> The same `stats` idea works for production configuration as well. See [the official documentation](https://webpack.js.org/configuration/stats/) for all the available options.

To benefit from these configuration parts, you need to connect them with *webpack.config.js* as in the complete code example below:

**webpack.config.js**

```javascript
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const webpack = require('webpack');
const merge = require('webpack-merge');

const parts = require('./webpack.parts');

const PATHS = {
  app: path.join(__dirname, 'app'),
  build: path.join(__dirname, 'build'),
};

const commonConfig = merge([
  {
    entry: {
      app: PATHS.app,
    },
    output: {
      path: PATHS.build,
      filename: '[name].js',
    },
    plugins: [
      new HtmlWebpackPlugin({
        title: 'Webpack demo',
      }),
    ],
  },
  parts.lintJavaScript({ include: PATHS.app }),
]);

const productionConfig = merge([
]);

const developmentConfig = merge([
  parts.devServer({
    // Customize host/port here if needed
    host: process.env.HOST,
    port: process.env.PORT,
  }),
]);

module.exports = (env) => {
  if (env === 'production') {
    return merge(commonConfig, productionConfig);
  }

  return merge(commonConfig, developmentConfig);
};
```

After this change, the build should behave the same way as before. This time, however, you have room to expand, and you don't have to worry about how to combine different parts of the configuration.

You can add more targets by expanding the *package.json* definition and branching at *webpack.config.js* based on the need. *webpack.parts.js* grows to contain specific techniques you can then use to compose the configuration.

T> Webpack 2 validates the configuration by default. If you make a mistake like a typo, it lets you know.

## The Benefits of Composing Configuration

Splitting configuration allows you to keep on expanding the setup. The biggest win is the fact that you can extract commonalities between different targets. You can also identify smaller configuration parts to compose. These configuration parts can be pushed to packages of their own to consume across projects.

Instead of duplicating similar configuration across multiple projects, you can manage configuration as a dependency now. As you figure out better ways to perform tasks, all your projects receive the improvements.

Each approach comes with its pros and cons. Composition-based approach myself is a good starting point. In addition to composition, it gives you a limited amount of code to scan through, but it's a good idea to check out how other people do it too. You can find something that works the best based on your tastes.

Perhaps the biggest problem is that with composition you need to know what you are doing, and it's possible you aren't going to get the composition right the first time around. But that's a software engineering problem that goes beyond webpack.

You can always iterate on the interfaces and find better ones. By passing in a configuration object instead of multiple arguments, you can change the behavior of a part without effecting its API. You can expose API as you need it.

T> If you have to support both webpack 1 and 2, you can perform branching based on version using `require('webpack/package.json').version` to detect it. After that, you have to set specific branches for each and merge.

## Configuration Layouts

In the book project, you push all of the configuration into two files: *webpack.config.js* and *webpack.parts*. The former contains higher level configuration while the latter lower level and isolates you from webpack specifics. Fortunately, the chosen approach allows more layouts, and you can evolve it further. Consider the following directions.

### Split per Configuration Target

If you split the configuration per target, you could end up with a file structure like this:

```bash
.
└── config
    ├── webpack.common.js
    ├── webpack.development.js
    ├── webpack.parts.js
    └── webpack.production.js
```

In this case, you would point to the targets through webpack `--config` parameter and `merge` common configuration at a target through `module.exports = merge(common, config);` kind of calls.

### Split Parts per Purpose

To add hierarchy to the way configuration parts are managed, you could decompose *webpack.parts.js* per category like this:

```bash
.
└── config
    ├── parts
    │   ├── devserver.js
    │   ├── fonts.js
    │   ├── images.js
    │   ├── index.js
    │   └── javascript.js
    └── ...
```

This arrangement would make it faster to find configuration related to a category. A good option would be to arrange the parts within a single file and use comments to split it up.

### Pushing Parts to Packages

Given all configuration is JavaScript, nothing prevents you from consuming it as a package or packages. It would be possible to package the shared configuration so that you can consume it across multiple projects. See the *Authoring Packages* chapter for further information on how to achieve this.

{pagebreak}

## Conclusion

Even though the configuration is technically the same as before, now you have room to grow it.

To recap:

* Given webpack configuration is JavaScript code underneath, there are many ways to manage it.
* You should choose a method to compose configuration that makes the most sense to you. [webpack-merge](https://www.npmjs.com/package/webpack-merge) was developed to provide a light approach for composition, but you can find many other options in the wild.
* Composition can enable configuration sharing. Instead of having to maintain a custom configuration per repository, you can share it across repositories this way. Using npm packages enables this. Developing configuration is close to developing any other code. This time, however, you codify your practices as packages.

The next parts of this book cover different techniques, and *webpack.parts.js* sees a lot of action as a result. The changes to *webpack.config.js* fortunately remain minimal.
