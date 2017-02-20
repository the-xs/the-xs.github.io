#Hot reload made easy with webpack
*by the-xs*

At some point, hot reload can help speed up the development when UI and states get complicated. In case needed, there is [post](http://stackoverflow.com/a/24587740/6013584) help understand it. So this could save you a lot of time when you want to debug a component with a bunch of state already loaded in the UI; HMR makes the state unchanged but just hot swap the changes you have made to your component.

There are different ways to provide HMR to your project; Some methods might not work for all situation. For example, the stateless functional compoenet might not be able to hot replaced with babel-preset-react-hmre:

```javascript
const Widget = () => <div>hello, word</div>
```

The sympton is console logs HMR is enabled, and components are updated, but browser content isn't updated. Troubleshooting HMR is quite tedious.

##Webpack 2 integration with react-hot-loader
Now we have [react-hot-loader](https://github.com/gaearon/react-hot-loader/pull/240); It's still in beta, but it's in a very good shape to work with your project and webpack has integrated it very well in its new version.

Let's start some normal setup with minimum needs:

```javascript
// get react
npm init -y
npm install --save react react-dom
```

```javascript
// get babel
npm install --save-dev babel-core babel-loader babel-preset-es2015 babel-preset-react babel-preset-stage-2
```

```javascript
// get webpack
npm install --save-dev webpack webpack-dev-server
```

Get react-hot-loader
```javascript
// latest beta as of now
npm install --save-dev react-hot-loader@3.0.0-beta.6
```

Place this line to package.json at the "script" section, so you can start a webpack-dev-server at port 3000.
```
"start": "webpack-dev-server --port 3000"
```

Edit the bable configuration: .babelrc

```javascript
{
  "presets": [
    ["es2015", {"modules": false}], //tree shaking
    "stage-2",
    "react"
  ],
  "plugins": [
    "react-hot-loader/babel"
  ]
}
```
Edit the webpack configuration: webpack.config.js

```javascript
const { resolve } = require('path');
const webpack = require('webpack');

const dist = resolve(__dirname, 'dist');
const src = resolve(__dirname, 'src');
const rootDir = '/';

module.exports = {
  devServer: {
    hot: true, //enable HMR
    contentBase: dist, //assume we use ./dist for output folder
    publicPath: rootDir,
  },

  plugins: [
    new webpack.HotModuleReplacementPlugin(),
    new webpack.NamedModulesPlugin(),
  ],

  devtool: 'inline-source-map',

  context: src,

  entry: [
    'react-hot-loader/patch', //active react-hot-loader
    'webpack-dev-server/client?http://localhost:3000',
    'webpack/hot/only-dev-server',
    './index.js',
  ],

  output: {
    filename: 'app.bundle.js',
    path: dist,
    publicPath: rootDir,
  },

  module: {
    rules: [
      {
        test: /\.(js|jsx)$/,
        use: ['babel-loader'],
        exclude: /node_modules/
      }
    ]
  }
}
```

The above webpack configuration suggests there is ./src folder for the sources, and ./dist folder for the output(transpiled) files.

Place a index.html into ./dist folder manually, or you can place it in ./src folder and have webpack configured to copy it over to ./dist folder.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
</head>
<body>
<div id="root"></div>
<script src="app.bundle.js"></script>
</body>
</html>
```

Last but not least, in ./src folder, the entry index.js has HMR module explicitly loaded. Pay attention to the last section of the code. If removed, changes to the source code would cause the page to refresh.

```javascript
/* ./src/index.js */
import React from 'react';
import ReactDOM from 'react-dom';

import { AppContainer } from 'react-hot-loader';
// AppContainer is a necessary wrapper component for HMR

import App from './app';

const render = (Component) => {
  ReactDOM.render(
    <AppContainer>
      <Component/>
    </AppContainer>,
    document.getElementById('root')
  );
};

render(App);

// Hot Module Replacement API
if (module.hot) {
  module.hot.accept('./app', () => {
    render(App)
  });
}

```

To test it, add an input box to the component. Type anything when page is loaded, and change the source code by editing the div content. 

```javascript
import React from 'react'

const App = () => (
  <div>
    <input />
    <h2>hello, world</h2>
  </div>
)

export default App;

```

**One more last piece: don't forget the distribution strategy. Disable/remove the HMR for end-user distribution.**

02/19/2017
@the-xs