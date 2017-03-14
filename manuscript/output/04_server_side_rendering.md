# Server Side Rendering

**Server Side Rendering** (SSR) is a technique that allows you to serve an initial payload with HTML, JavaScript, CSS, and even application state. You serve a fully rendered HTML page that would make sense even without JavaScript enabled. In addition to providing potential performance benefits, this can help with Search Engine Optimization (SEO).

Even though the idea does not sound that special, there is a technical cost involved, and you can find sharp corners. The approach was popularized by React. Since then frameworks encapsulating the tricky bits, such as [Next.js](https://www.npmjs.com/package/next), have appeared. [isomorphic-webpack](https://www.npmjs.com/package/isomorphic-webpack) is a good example of a solution designed on top of webpack.

To demonstrate SSR, you can use webpack to compile a client-side build that then gets picked up by a server that renders it using React following the principle. Demonstration like this is enough to understand how it works and also where the problems begin.

## Setting Up Babel with React

The *Loading JavaScript* chapter covers the essentials of using Babel with webpack. There's setup that is particular to React you should perform, though. Given most of React projects rely on [JSX](https://facebook.github.io/jsx/) format, you have to enable it through Babel.

To get React, and particularly JSX, work with Babel, install the preset first:

```bash
npm install babel-preset-react --save-dev
```

Connect the preset with Babel configuration as follows:

**.babelrc**

```json
{
  ...
  "presets": [
leanpub-start-insert
    "react",
leanpub-end-insert
    ...
  ]
}
```

## Configuring React with ESLint

Using React with ESLint and JSX requires extra work as well. [eslint-plugin-react](https://www.npmjs.com/package/eslint-plugin-react) does a part of the work, but also ESLint configuration is needed.

Install *eslint-plugin-react* to get started:

```bash
npm install eslint-plugin-react --save-dev
```

The suggested minimum configuration is as follows:

**.eslintrc.js**

```javascript
module.exports = {
  ...
leanpub-start-delete
  "extends": "eslint:recommended",
leanpub-end-delete
leanpub-start-insert
  "extends": ["eslint:recommended", "plugin:react/recommended"],
leanpub-end-insert
  "parser": "babel-eslint",
  "parserOptions": {
    "sourceType": "module",
    "allowImportExportEverywhere": true,

leanpub-start-insert
    // Enable JSX
    "ecmaFeatures": {
      "jsx": true,
    },
leanpub-end-insert
  },
leanpub-start-insert
  "plugins": [
    "react",
  ],
leanpub-end-insert
  ...
};
```

You can enable more specific rules based on your liking, but the `plugin:react/recommended` gives a good starting point. It's important to remember to enable JSX at the `parserOptions` as well.

## Setting Up a React Demo

To make sure the project has the dependencies in place, install React and [react-dom](https://www.npmjs.com/package/react-dom). The latter package is needed to render the application to the DOM.

```bash
npm install react react-dom --save
```

Next, the React code needs a small entry point. If you are on the browser side, you should mount `Hello world` `div` to the document. To prove it works, clicking it should give a dialog with a "hello" message. On server-side the React component is returned so the server can pick it up:

**app/ssr.js**

```javascript
const React = require('react');
const ReactDOM = require('react-dom');

const SSR = <div onClick={() => alert('hello')}>Hello world</div>;

// Render only in the browser, export otherwise
if (typeof document === 'undefined') {
  module.exports = SSR;
} else {
  ReactDOM.render(SSR, document.getElementById('app'));
}
```

You are still missing webpack configuration to turn this file into something the server can pick up.

W> Given ES6 style imports and CommonJS exports cannot be mixed, the entry point was written in CommonJS style.

## Configuring Webpack

To keep things nice and tidy, it's possible to push the demo configuration to a file of its own. A lot of the work has been done already. Given you have to consume the same output from multiple environments, using UMD as the library target makes sense:

**webpack.ssr.js**

```javascript
const path = require('path');
const merge = require('webpack-merge');

const parts = require('./webpack.parts');

const PATHS = {
  build: path.join(__dirname, 'static'),
  ssrDemo: path.join(__dirname, 'app', 'ssr.js'),
};

module.exports = merge([
  {
    entry: {
      index: PATHS.ssrDemo,
    },
    output: {
      path: PATHS.build,
      filename: '[name].js',
      libraryTarget: 'umd',
    },
  },
  parts.loadJavaScript({ include: PATHS.ssrDemo }),
]);
```

To make it convenient to generate a build, add a helper script.

**package.json**

```json
...
"scripts": {
leanpub-start-insert
  "build:ssr": "webpack --config webpack.ssr.js",
leanpub-end-insert
  ...
},
...
```

If you build the SSR demo (`npm run build:ssr`), you should see a new file at *./static/index.js*. The next step is to set up a server to render it.

## Setting Up a Server

To keep things clear to understand, you can set up a standalone Express server that picks up the generated bundle and renders it following the SSR principle. Install Express first:

```bash
npm install express --save-dev
```

Then, to get something running, implement a server as follows:

**server.js**

```javascript
const express = require('express');
const { renderToString } = require('react-dom/server');

const SSR = require('./static');

server(process.env.PORT || 8080);

function server(port) {
  const app = express();

  app.use(express.static('static'));
  app.get('/', (req, res) => (
    res.status(200).send(
      renderMarkup(renderToString(SSR))
    )
  ));

  app.listen(port);
}

function renderMarkup(html) {
  return `<!DOCTYPE html>
<html>
  <head>
    <title>Webpack SSR Demo</title>
    <meta charset="utf-8" />
  </head>
  <body>
    <div id="app">${html}</div>
    <script src="./index.js"></script>
  </body>
</html>`;
}
```

If you run the server now (`node ./server.js`) and navigate below `http://localhost:8080`, you should see something familiar:

![Hello world](images/hello_01.png)

Even though there is a basic React application running now, it's difficult to develop. If you try to modify the code, nothing happens. This can be solved running webpack in a multi-compiler mode as earlier in this book. Another option is to run webpack in **watch mode** against the current configuration and set up a watcher for the server. You learn the latter setup next.

T> If you want to debug output from the server, set `export DEBUG=express:application`.

T> The references to the assets generated by webpack could be written automatically to the server side template if you wrote a manifest as discussed in the *Separating a Manifest* chapter.

W> If you get a linting warning like `warning  'React' is defined but never used  no-unused-vars`, make sure the ESLint React plugin has been enabled and its default preset is in use.

## Watching SSR Changes and Refreshing the Browser

The first portion of the problem is fast to solve. Run `npm run build:ssr -- --watch` in a terminal. That forces webpack to run in a watch mode. It would be possible to wrap this idea within an npm script for convenience, but this is enough for this demo.

The remaining part is harder than what was done so far. How to make the server aware of the changes and how to communicate the changes to the browser?

[browser-refresh](https://www.npmjs.com/package/browser-refresh) can come in handy as it solves both of the problems. Install it first:

```bash
npm install browser-refresh --save-dev
```

The client portion requires two small changes to the server code:

**server.js**

```javascript
...

server(process.env.PORT || 8080);

function server(port) {
  ...

leanpub-start-delete
  app.listen(port);
leanpub-end-delete
leanpub-start-insert
  app.listen(port, () => process.send && process.send('online'));
leanpub-end-insert
}

function renderMarkup(html) {
  return `<!DOCTYPE html>
<html>
  ...
  <body>
    ...
leanpub-start-insert
    <script src="${process.env.BROWSER_REFRESH_URL}"></script>
leanpub-end-insert
  </body>
</html>`;
}
```

The first change tells the client that the application is online and ready to go. The latter change attaches the client script to the output. *browser-refresh* manages the environment variable in question.

Run `node_modules/.bin/browser-refresh ./server.js` in another terminal and open the browser at `http://localhost:8080` as earlier to test the setup. Remember to have webpack running in the watch mode at another terminal. If everything went right, any change you make to the demo client script (*app/ssr.js*) should show up in the browser or cause a failure at the server.

If the server crashes, it loses the WebSocket connection. You have to force a refresh in the browser in this case. If the server was managed through webpack as well, the problem could have been avoided.

To prove that SSR works, check out the browser inspector. You should see something familiar there:

![SSR output](images/ssr.png)

Instead of a `div` where to mount an application, you can see all related HTML there. It's not much in this particular case, but it's enough to showcase the approach.

T> The current implementation could be refined further by implementing a production mode for the server that would skip injecting the browser refresh script at a minimum.

T> The server could inject initial data payload to the generated HTML. Doing this would avoid queries on the client-side.

## Open Questions

Even though the demo illustrates the basic idea of SSR, it still leaves open questions:

* How to deal with styles? Vanilla Node doesn't understand CSS related imports by design.
* How to deal with anything else than JavaScript? If the server side is processed through webpack, this is less of an issue as you can patch it at webpack.
* How to run the server through something else than Node? One option would be to wrap the Node instance in a service you then run through your host environment. Ideally, the results would be cached, and you can find more specific solutions for this particular per platform.

Questions like these are the reason why solutions such as [isomorphic-webpack](https://www.npmjs.com/package/isomorphic-webpack) or [Next.js](https://github.com/zeit/next.js) exist. They have been designed to solve SSR-specific problems like these.

T> Routing is a big problem of its own solved by frameworks like Next.js. Patrick Hund [discusses how to solve it with React and React Router 4](https://ebaytech.berlin/universal-web-apps-with-react-router-4-15002bb30ccb).

## Conclusion

SSR is a big topic on its own. It comes with a technical challenge, and for this reason, specific solutions have appeared around it. Webpack is a good fit for SSR setups. The idea can be adapted to other purposes.

To recap:

* The idea of **Server Side Rendering** is to provide more for the browser to render initially. Instead of waiting for the JavaScript to load, it can display markup instantly.
* Server Side Rendering also allows you to pass initial payload of data to the client. This can help you to avoid unnecessary queries to the server.
* Webpack can manage the client side portion of the problem. It can be used to generate the server as well if more integrated solution is required. Abstractions, such as Next.js, hide these details.
* Server Side Rendering does not come without a cost. It leads to new problems as you need better approaches for dealing with aspects, such as styling or routing. The server and the client environment differ in important manners, so code has to be written so that it does not rely on platform-specific features too much.
