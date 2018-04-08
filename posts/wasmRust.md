# Setting up WASM + Rust + WebPack + React + ES6/7

Welcome! 

For those who are new to [WebAssembly](http://webassembly.org) and [Rust](https://www.rust-lang.org/en-US/), follow along to see how to get WebAssembly and Rust working happily together!

We will be using [Webpack 4](https://webpack.js.org) as our bundler as well as [React](https://reactjs.org) for our frontend. How we set these up will be discussed in the next section.

To get started, we should first install Rust via the helpful rustup.sh:

```
curl https://sh.rustup.rs -sSf | sh
```

We want to default the nightly toolchain, and also install the wasm64-unknown-unknown toolchain as well:

```
rustup install nightly
rustup default nightly
rustup target add wasm32-unknown-unknown
```

Rust has native support within its compiler to compile Rust code straight into WebAssembly, all without emscripten!

## Getting ES6 set up with ES7 features

We will be using [Yarn](https://yarnpkg.com/lang/en/docs/install/) instead of npm, but the workflow should be the same in both, just different command syntax.

### Init A New Node.js Project:

```
$ yarn init
```

### Install Babel presets

To transpile ES6 JavaScript into ES2015 for the browser, we need to use [babel](https://babeljs.io):

```
$ yarn add babel-loader babel-polyfill babel-preset-env babel-preset-react babel-preset-stage-0
```

Note: we also need to create a .babelrc file to use these presets:
```
$ touch .babelrc
```
In .babelrc put:
```
{
  "presets": ["env", "react", "stage-0"]
}
```

### Install React

```
$ yarn add react react-dom
```

We also need to use the very helpful tool [wasm-bindgen](https://github.com/rustwasm/wasm-bindgen). The gist of what we need to install is as follows:

```
$ cargo install wasm-bindgen-cli
```

This will allow us to have higher level interactions between JS and Rust, yay!

### Finally, Install Webpack Dependencies

```
$ yarn global add webpack
$ yarn global add webpack-cli
$ yarn global add webpack-dev-server
```

# Meat + Potatoes

Now for the fun stuff, lets create a new Rust project:

```
$ cargo init --lib
```

### Our Cargo.toml

```
[package]
name = "our-cool-wasm-app"
version = "0.1.0"
authors = ["Your Name Here"]

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.1"
```

### The JS

Lets create our JS files:

```
$ touch app.jsx
$ touch webpack.config.js
```

### The Webpack Config

We need to tell webpack to transpile our .jsx files, so our config should look like this (our code is served out of the dist/ directory, so don't change anything in here!):

```
const path = require('path');

module.exports = {
  entry: ['./app.jsx'],
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "index.js",
  },
  mode: "development",
  module: {
    rules: [
      {
        test: /\.jsx$/,
        exclude: /node_modules/,
        use: [
          "babel-loader",
        ],
      }
    ]
  },
};
```

### Our Frontend

We are using the ES7 feature async/await to load the wasm bundle, so we have to polyfill our app.jsx, its ugly, but it works. At the top of app.jsx put:

```javascript
// app.jsx
/* We can only have one instance of babel-polyfill*/
global._babelPolyfill = false;
import 'babel-polyfill'
``` 

After we import babel-polyfill, lets hookup react.

### The React Boilerplate

```javascript
// previous imports

import React, { Component } from 'react'
import ReactDOM from 'react-dom'

```

Now we can create our App class:

```javascript
/* app.jsx */
// previous imports 
class App extends Component {
    constructor(props) {
      super(props);
      this.state = {
        wasm: null /* We will store our wasm module here */
      }
    }

    /* Our method to load the wasm module */
    async handleWASM() {
      const wasm_module = await import('./rust_sandbox');
      if (this.mounted) {
        this.setState({wasm: wasm_module});
      }
    }
  

    componentDidMount() {
      this.mounted = true;
      this.handleWASM();
    }

    componentWillUnmount() {
      this.mounted = false;
    }

    /* The methods we call here will make sense in the next section */
    render() {
      return this.state.wasm ? (
        <div>
          <p>Rust String: {this.state.wasm.hello_world()}</p>
          <p>Rust Struct: ContainsData {"{"} {
            this.state.wasm.ContainsData.new().add(10)
          } {"}"}</p>
        </div>
      ) : (<div>Loading...</div>)
    }
}

/* Finally we render our App to the DOM */
ReactDOM.render(<App />, document.getElementById("application"));

```
### The Rust Library

In src/lib.rs put the following code, note the use of the #[wasm_bindgen] decorator we mentioned earlier:
```rust
#![feature(proc_macro, wasm_custom_section, wasm_import_module)]

extern crate wasm_bindgen;

use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern {
    fn alert(s: &str);
}

#[wasm_bindgen]
pub fn greet(name: &str) {
    alert(&format!("Hello, {}!", name));
}

#[wasm_bindgen]
pub fn hello_world() -> String {
  "Hello World!".to_owned()
}

// Strings can both be passed in and received
#[wasm_bindgen]
pub fn concat(a: &str, b: &str) -> String {
    let mut a = a.to_string();
    a.push_str(b);
    return a
}

#[wasm_bindgen]
pub struct ContainsData {
    contents: u32,
}

#[wasm_bindgen]
impl ContainsData {
    pub fn new() -> ContainsData {
        ContainsData { contents: 0 }
    }

    pub fn add(&mut self, amt: u32) -> u32 {
        self.contents += amt;
        return self.contents
    }
}
```

### The HTML

Now lets create an index.html file:

```html
<html>
  <head>
    <meta content="text/html;charset=utf-8" http-equiv="Content-Type"/>
  </head>
  <body>
    <div id="application"></div>
    <script src='./index.js'></script>
  </body>
</html>
```

### Finally, We Build And Run Our App

In package.json, add a "scripts" section with the following contents:

```
"scripts": {
    "build": "webpack --progress --colors --hot",
    "serve": "yarn assemble && yarn bindgen && yarn build && webpack-dev-server dist/index.js",
    "bindgen": "wasm-bindgen target/wasm32-unknown-unknown/release/rust_sandbox.wasm --out-dir .",
    "assemble": "cargo +nightly build --release --target wasm32-unknown-unknown"
  },
```

Then in terminal in the root of our project, run this command:
```
yarn serve
```

If everything goes well, Rust will be assembled into a .wasm file, bindings will be generated for it, and it will be transpiled into dist/index.js along with our ES6 Javascript. 

The bundle should now be served at:

http://localhost:8000

Thanks for following along! 

The repo for this project is at: 

https://github.com/BrantleighBunting/wasm-sandbox

- [Markdown Test Page](#lorem-ipsum)
