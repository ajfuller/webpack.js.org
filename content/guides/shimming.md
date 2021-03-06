---
title: Shimming
sort: 17
contributors:
  - pksjce
  - jhnns
  - simon04
---

`webpack` as a module bundler can understand modules written as ES2015 modules, CommonJS or AMD. But many times, while using third party libraries, we see that they expect dependencies which are global aka `$` for `jquery`. They might also be creating global variables which need to be exported. Here we will see different ways to help webpack understand these __broken modules__.

## Prefer unminified CommonJS/AMD files over bundled `dist` versions.

Most modules link the `dist` version in the `main` field of their `package.json`. While this is useful for most developers, for webpack it is better to alias the src version because this way webpack is able to optimize dependencies better. However in most cases `dist` works fine as well.

``` javascript
// webpack.config.js

module.exports = {
    ...
    resolve: {
        alias: {
            jquery: "jquery/src/jquery"
        }
    }
};
```

## `provide-plugin`
The [`provide-plugin`](/plugins/provide-plugin) makes a module available as a variable in every other module required by `webpack`. The module is required only if you use the variable.
Most legacy modules rely on the presence of specific globals, like jQuery plugins do on `$` or `jQuery`. In this scenario, you can configure webpack to prepend `var $ = require(“jquery”)` every time it encounters the global `$` identifier.

```javascript
module.exports = {
  module: [
    new webpack.ProvidePlugin({
      $: 'jquery',
      jQuery: 'jquery'
    })
  ]
};
```

## `imports-loader`

[`imports-loader`](/loaders/imports-loader/) inserts necessary globals into the required legacy module.
For example, Some legacy modules rely on `this` being the `window` object. This becomes a problem when the module is executed in a CommonJS context where `this` equals `module.exports`. In this case you can override `this` using the `imports-loader`.

**webpack.config.js**
```javascript
module.exports = {
  module: {
    rules: [{
      test: require.resolve("some-module"), 
      use: 'imports-loader?this=>window'
    }]
  }
};
```

There are modules that support different [module styles](/concepts/modules), like AMD, CommonJS and legacy. However, most of the time they first check for `define` and then use some quirky code to export properties. In these cases, it could help to force the CommonJS path by setting `define = false`:

**webpack.config.js**
```javascript
module.exports = {
  module: {
    rules: [{
      test: require.resolve("some-module"), 
      use: 'imports-loader?define=>false'
    }]
  }
};
```

## `exports-loader`

Let's say a library creates a global variable that it expects it's consumers to use. In this case we can use [`exports-loader`](/loaders/exports-loader/), to export that global variable in CommonJS format. For instance, in order to export `file` as `file` and `helpers.parse` as `parse`:

**webpack.config.js**
```javascript
module.exports = {
  module: {
    rules: [{
      test: require.resolve("some-module"), 
      use: 'exports-loader?file,parse=helpers.parse'
      // adds below code the the file's source:
      //  exports["file"] = file;
      //  exports["parse"] = helpers.parse;
    }]
  }
};
```

## `script-loader`

The [script-loader](/loaders/script-loader/) evaluates code in the global context, just like you would add the code into a `script` tag. In this mode every normal library should work. require, module, etc. are undefined.

W> The file is added as string to the bundle. It is not minimized by `webpack`, so use a minimized version. There is also no dev tool support for libraries added by this loader.

Assuming you have a `legacy.js` file containing …
```javascript
GLOBAL_CONFIG = {};
```

… using the `script-loader` …

```javascript
require('script-loader!legacy.js');
``` 

… basically yields:

```javascript
eval("GLOBAL_CONFIG = {};");
```

## `noParse` option

When there is no AMD/CommonJS version of the module and you want to include the `dist`, you can flag this module as [`noParse`](/configuration/module/#module-noparse). Then `webpack` will just include the module without parsing it, which can be used to improve the build time. 

W> Any feature requiring the AST, like the `ProvidePlugin`, will not work.

```javascript
module.exports = {
  module: {
    noParse: /jquery|backbone/
  }
};
```
