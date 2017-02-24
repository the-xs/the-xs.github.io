#Tree Shaking with Webpack 2

Tree shaking is a very good technique to reduce the bundle file size. When we bundle the Javascript files, we want to remove the unused code just like shaking a tree to remove the dead branches.

When using the es6 to write the code, bundling is a little tricky. Webpack 2 supports native es6 now, but to shaking off the dead branches, babel transpile is still needed.

##Modules with webpack 2 es6 native suppport

**Some initial setup**

```
npm init -y
npm i --save-dev webpack
mkdir src
mkdir dist
touch ./src/my-module.js
touch ./src/index-my-module.js
```

**Add some code to the files**

```Javascript
// ./src/my-module.js
export const Sin = (x) => Math.Sin(x);

export const Sqrt = (x) => Math.sqrt(x);

export const Log = (x) => Math.log(x);

// ./src/index-my-module.js
import { Sin } from './my-module';

Console.log(Sin(2));
```

**Webpack configuration**

```Javascript
// ./webpack.my-module.js
module.exports = {
  entry: './src/index-my-module.js',
  output: { filename: 'my-module-test.bundle.js', path: 'dist' },
}
```

**Build it**

```
./node_module/.bin/webpack --config webpack.my-module.js
```

Check the bundle file, webpack now recognizes the dead branches. There are some code looks like this:

```Javascript
"use strict";
const Sin = (x) => Math.Sin(x);
/* harmony export (immutable) */ __webpack_exports__["a"] = Sin;


const Sqrt = (x) => Math.sqrt(x);
/* unused harmony export Sqrt */


const Log = (x) => Math.log(x);
/* unused harmony export Log */
```

**Let's shake it off**

```
node_modules/.bin/webpack --config webpack.my-module.js -p

//oops, here is the output
Hash: f72f2be6623bd5893b0b
Version: webpack 2.2.1
Time: 84ms
                   Asset     Size  Chunks             Chunk Names
my-module-test.bundle.js  3.15 kB       0  [emitted]  main
   [0] ./src/my-module.js 120 bytes {0} [built]
   [1] ./src/index-my-module.js 56 bytes {0} [built]

ERROR in my-module-test.bundle.js from UglifyJs
SyntaxError: Unexpected token: operator (>) [my-module-test.bundle.js:74,17]

```


##Webpack 2 + babel for the real world

Even though webpack 2 supports the native es6, we still need to transform the es6 to support all browsers for "production" ready code.

We need to add babel-loader:

```
npm i --save-dev babel-core babel-loader babel-preset-es2015
```

For babel, we now have better support for tree shaking as well, by add ```{modules: false}``` to not use the babel-plugin-transform-es2015-modules-commonjs. To do proper tree shaking, the modules have to be static structure, but ```CommonJS``` module is not a static one.

```Javascript
module.exports = {
  entry: './src/index-my-module.js',
  output: { filename: 'my-module-test.bundle.js', path: 'dist' },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        loader: 'babel-loader',
        options: { 
          presets: [ 
            [ 'es2015', { modules: false } ] //tree shaking
          ] 
        }
      }
    ]
  },
}
```

**Shaking it off again after the fix**

```node_modules/.bin/webpack --config webpack.my-module.js -p
Time: 462ms
                   Asset       Size  Chunks             Chunk Names
my-module-test.bundle.js  711 bytes       0  [emitted]  main
   [0] ./src/my-module.js 184 bytes {0} [built]
   [1] ./src/index-my-module.js 56 bytes {0} [built]
```

We are not seeing ```Log``` or ```Sqrt``` components any more in the bundle file.

##React and tree shaking can work together too

```
// setup
npm i --save react react-dom
npm i --save-dev babel-preset-react html-webpack-plugin
touch ./src/index-react.js
touch ./src/components.js
```

```Javascript
// ./src/components.js
import React from 'react';

export const WidgetUseful = () => <div>this is very useful widget</div>;

export const WidgetUseless = () => <Button>this is old useless button</Button>;

export default "I am a little teapot";

```

```Javascript
// ./src/index-react.js
import React from 'react';
import ReactDom from 'react-dom';
import { WidgetUseful } from './components'; 

ReactDom.render(
  <WidgetUseful />, document.body
)

```

```Javascript
// ./webpack.react.js
const HtmlWebpackPlugin = require('html-webpack-plugin');
module.exports = {
  entry: './src/index-react.js',
  output: { filename: 'react.bundle.js', path: 'dist' },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        loader: 'babel-loader',
        options: { 
          presets: [ 
            [ 'es2015', { modules: false } ],
            'react',
          ] 
        }
      }
    ]
  },
  plugins: [ new HtmlWebpackPlugin({ title: 'Tree-shaking' }) ]
}
```

**Tree shaking with react**

```
./node_modules/.bin/webpack --config webpack.react.js
// check and search the bundle file, unused code is identified.

./node_modules/.bin/webpack --config webpack.react.js -p
// turn on switch -p, the code is removed
```

@xs
