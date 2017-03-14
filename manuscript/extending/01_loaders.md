# Extending with Loaders

As you have seen so far, loaders are one of the building blocks of webpack. If you want to load an asset, you most likely need to set up a matching loader definition. Even though there are a lot of [available loaders](https://webpack.js.org/loaders/), it's possible you are missing one fitting your purposes.

You learn to develop a couple of small loaders next. But before that, it's good to understand how to debug them in isolation.

T> If you want a good starting point for a standalone loader or plugin project, consider using [webpack-defaults](https://github.com/webpack-contrib/webpack-defaults). It provides an opinionated starting point that comes with linting, testing, and other goodies.

## Debugging Loaders with *loader-runner*

[loader-runner](https://www.npmjs.com/package/loader-runner) allows you to run loaders without webpack making it a good tool for learning more about loader development. Install it first:

```bash
npm install loader-runner --save-dev
```

To have something to test with, set up a loader that returns twice what's passed to it like this:

**loaders/demo-loader.js**

```javascript
module.exports = (input) => input + input;
```

Set up a file to process:

**demo.txt**

```
foobar
```

There's nothing webpack specific in the code yet. The next step is to run the loader through *loader-runner*:

**run-loader.js**

```javascript
const fs = require('fs');
const { runLoaders } = require('loader-runner');

runLoaders({
  resource: './demo.txt',
  loaders: [
    path.resolve(__dirname, './loaders/demo-loader'),
  ],
  readResource: fs.readFile.bind(fs),
},
(err, result) => err ?
  console.error(err) :
  console.log(result)
);
```

If you run the script now (`node run-loader.js`), you should see output like this:

```javascript
{ result: [ 'foobar\nfoobar\n' ],
  resourceBuffer: <Buffer 66 6f 6f 62 61 72 0a>,
  cacheable: true,
  fileDependencies: [ './demo.txt' ],
  contextDependencies: [] }
```

The output tells the `result` of the processing, the resource that was processed as a buffer, and other meta information. This is enough to develop more complicated loaders.

T> If you want to capture the output to a file, use either `fs.writeFileSync('./output.txt', result.result)` or its asynchronous version as discussed in [Node documentation](https://nodejs.org/api/fs.html).

T> It's possible to refer to loaders installed to the local project by name instead of resolving a full path to them. Example: `loaders: ['raw-loader']`.

## Implementing an Asynchronous Loader

Even though you can implement a lot of loaders using the synchronous interface, there are times when asynchronous calculation is required. Wrapping a third party package as a loader can force you to this.

The example above can be adapted to asynchronous form by using webpack specific API through `this.async()`. Webpack sets this and the function returns a callback following Node conventions (error first, result second). Tweak as follows:

**loaders/demo-loader.js**

```javascript
module.exports = function(input) {
  const callback = this.async();

  callback(null, input + input);
};
```

W> Given webpack injects its API through `this`, the shorter function form (`() => ...`) cannot be used here.

Running the demo script (`node run-loader.js`) again should give exactly the same result as before. To raise an error during execution, try the following:

**loaders/demo-loader.js**

```javascript
module.exports = function(input) {
  const callback = this.async();

  callback(new Error('Demo error'));
};
```

The result should contain `Error: Demo error` with a stack trace showing where the error originates.

## Returning Only Output

Loaders can be used to output code. You could have implementation like this:

**loaders/demo-loader.js**

```javascript
module.exports = function() {
  return 'foobar';
};
```

But what's the point? You can pass to loaders through webpack entries. Instead of pointint to pre-existing files as you would in majority of the cases, you could pass to a loader that generates code dynamically. Even though a special case, it's good to be aware of the technique.

T> If you want to return `Buffer` output, you will have to set `module.exports.raw = true` as webpack expects a string by default. The flag overrides this default behavior. You need this if you wrap for example [node-sass](https://www.npmjs.com/package/node-sass) as a loader.

## Writing Files

Loaders, like [file-loader](https://www.npmjs.com/package/file-loader), emit files. Webpack provides a single method, `this.emitFile`, for this. Given *loader-runner* does not implement it, you have to mock it:

**run-loader.js**

```javascript
const fs = require('fs');
const { runLoaders } = require('loader-runner');

runLoaders({
  resource: './demo.txt',
  loaders: [
    path.resolve(__dirname, './loaders/demo-loader'),
  ],
leanpub-start-insert
  context: {
    emitFile: () => {},
  },
leanpub-end-insert
  readResource: fs.readFile.bind(fs),
},
(err, result) => err ?
  console.error(err) :
  console.log(result)
);
```

To implement the essential idea of *file-loader*, you have to do two things: emit the file and return path to it. You could implement it like this:

**loaders/demo-loader.js**

```javascript
const loaderUtils = require('loader-utils');

module.exports = function(content) {
  const url = loaderUtils.interpolateName(
    this, '[hash].[ext]', { content }
  );

  this.emitFile(url, content);

  const filePath = `__webpack_public_path__ + ${JSON.stringify(url)};`;

  return `export default ${filePath}`;
};
```

Webpack provides two additional `emit` methods: `this.emitWarning` and `this.emitError`. Both accept a string and can be used to tell the user something unexpected happened. They should be used over `console` based alternatives. As with `this.emitFile`, you have to mock them for *loader-runner* to work.

The next question is, how to pass file name to the loader.

## Passing Options to Loaders

To demonstrate passing options, the runner needs a small tweak:

**run-loader.js**

```javascript
...

runLoaders({
  resource: './demo.txt',
  loaders: [
leanpub-start-delete
    path.resolve(__dirname, './loaders/demo-loader')
leanpub-end-delete
leanpub-start-insert
    {
      loader: path.resolve(__dirname, './loaders/demo-loader'),
      options: {
        name: 'demo.[ext]',
      },
    },
leanpub-end-insert
  ],
  ...
},
...
);
```

To capture the option, you need to use [loader-utils](https://www.npmjs.com/package/loader-utils). It has been designed to parse loader options and queries. Install it:

```bash
npm install loader-utils --save-dev
```

To connect it the loader, set it to capture `name` and pass it through webpack's interpolator:

**loaders/demo-loader.js**

```javascript
const loaderUtils = require('loader-utils');

module.exports = function(content) {
leanpub-start-insert
  const { name } = loaderUtils.getOptions(this);
leanpub-end-insert
  const url = loaderUtils.interpolateName(
leanpub-start-delete
    this, '[hash].[ext]', { content }
leanpub-end-delete
leanpub-start-insert
    this, name, { content }
leanpub-end-insert
  );

  this.emitFile(url, content);

  const filePath = `__webpack_public_path__ + ${JSON.stringify(url)};`;

  return `export default ${filePath}`;
};
```

After running (`node ./run-loader.js`), you should see something like this:

```javascript
{ result: [ 'export default __webpack_public_path__ + "demo.txt";' ],
  resourceBuffer: <Buffer 66 6f 6f 62 61 72 0a>,
  cacheable: true,
  fileDependencies: [ './demo.txt' ],
  contextDependencies: [] }
```

You can see that the result matches what the loader should have returned. You can try to pass more options to the loader or use query parameters to see what happens with different combinations.

T> It's a good idea to validate options and rather fail hard than silently if the options aren't what you expect. [schema-utils](https://www.npmjs.com/package/schema-utils) has been designed for this purpose.

## Pitch Loaders

Webpack evaluates loaders in two phases: pitching and running. If you are used to web event semantics, these map to capturing and bubbling. The idea is that webpack allows you to intercept execution during the pitching (capturing) phase. It goes through the loaders left to right first like this and after that it executes them from right to left.

A pitch loader allows you shape the request and even terminate it. Set up an example as follows:

**loaders/pitch-loader.js**

```javascript
const loaderUtils = require('loader-utils');

module.exports = function(input) {
  const { text } = loaderUtils.getOptions(this);

  return input + text;
};
module.exports.pitch = function(
  remainingRequest, precedingRequest, input
) {
  console.log(`
Remaining request: ${remainingRequest}
Preceding request: ${precedingRequest}
Input: ${JSON.stringify(input, null, 2)}
  `);

  return 'pitched';
};
```

To connect it to the runner, add it to the loader definition:

**run-loader.js**

```javascript
const fs = require('fs');
const path = require('path');
const { runLoaders } = require('loader-runner');

runLoaders({
  resource: './demo.txt',
  loaders: [
    {
      loader: path.resolve(__dirname, './loaders/demo-loader'),
      options: {
        name: 'demo.[ext]',
      },
    },
leanpub-start-insert
    path.resolve(__dirname, './loaders/pitch-loader'),
leanpub-end-insert
  ],
  context: {
    emitFile: () => {},
  },
  readResource: fs.readFile.bind(fs),
},
(err, result) => err ?
  console.error(err) :
  console.log(result)
);
```

If you run (`node ./run-loader.js`) now, the pitch loader should log intermediate data and intercept the execution:

```javascript
Remaining request: ./demo.txt
Preceding request: .../webpack-demo/loaders/demo-loader?{"name":"demo.[ext]"}
Input: {}

{ result: [ 'export default __webpack_public_path__ + "demo.txt";' ],
  resourceBuffer: null,
  cacheable: true,
  fileDependencies: [],
  contextDependencies: [] }
```

Although webpack caches loaders by default unless they set `this.cacheable(false)`, writing a caching loader can be a good exercise as it helps you to understand how loader stages can work together. The example below shows how to achieve this (courtesy of Vladimir Grenaderov):

```
const cache = new Map();

module.exports = function(content) {
  // Calls only once for given resourcePath
  const callbacks = cache.get(this.resourcePath);
  callbacks.forEach(callback => callback(null, content));

  cache.set(this.resourcePath, content);

  return content;
};
module.exports.pitch = function() {
  if (cache.has(this.resourcePath)) {
    const item = cache.get(this.resourcePath);

    if (item instanceof Array) {
      // Load to cache
      item.push(this.async());
    } else {
      // Hit cache
      return item;
    }
  } else {
    // Missed cache
    cache.set(this.resourcePath, []);
  }
};
```

A pitch loader can be used to attach metadata to the input to use later. In this example, cache was constructed during the pitching stage and it was accessed during normal execution.

T> The [official documentation](https://webpack.js.org/api/loaders/) covers the loader API in detail. You can see all fields available through `this` there.

W> Although using *loader-runner* can be convenient for developing and testing loaders, it's a good idea to implement integration tests that run it against webpack itself. Node and webpack environments have subtle differences making this essential.

## Conclusion

Writing loaders is fun in the sense that they describe transformations from a format to another. Often you can figure out how to achieve something specific by either studying either the API documentation or the existing loaders.

To recap:

* *loader-runner* is a valuable tool for understanding how loaders work. Use it for debugging how loaders work.
* Webpack **loaders** accept input and produce output based on it.
* Loaders can be either synchronous or asynchronous. In the latter case, you should use `this.async()` webpack API to capture the callback exposed by webpack.
* If you want to generate code dynamically for webpack entries, that's where loaders can come in handy. A loader does not have to accept input. It's acceptable that it returns only output in this case.
* Use **loader-utils** to parse possible options passed to a loader and consider validating them using **schema-utils**.
* Pitching stage complements the default behavior allowing you to intercept and to attach metadata.

You learn to write plugins in the next chapter. Plugins allow you to intercept webpack's execution process and they can be combined with loaders to develop more advanced functionality.
