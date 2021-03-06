---
layout: default
title: Resources
nav_order: 4
permalink: /resources
---

# Resources

Resources are the gulp vinyl files you throw to `dumber`.

Use again the example showed in [Get Started](get-started).

```js
const dumber = require('gulp-dumber');
const dr = dumber({/* ... */});

function build() {
  return merge2(
    gulp.src('src/**/*.json'),
    gulp.src('src/**/*.html'),
    gulp.src('src/**/*.js').pipe(babel()),
    gulp.src('src/**/*.scss').pipe(sass())
  )
  // Here is dumber doing humble bundling.
  // The extra call `dr()` is designed to cater watch mode.
  .pipe(dr())
  .pipe(gulp.dest('dist'));
}
```

Those json, html, js, css (gulp-sass turns scss vinyl file to css vinyl file) are resources.

`dumber` treats following resources specially:

## 1. js files

`dumber` transforms `.js` files into AMD modules.

`dumber` understands JavasScript code in either CommonJS, ESM (esnext), AMD, or UMD format.

`dumber` is very dumb, it doesn't understand `.ts` (TypeScript) or `.coffee` (CoffeeScript) files. You need to use gulp-typescript or gulp-coffee to transpile those files into `.js` vinyl files before sending to `dumber`.

Not required by `dumber`, but we recommend TypeScript users to turn on `esModuleInterop` to avoid possible headaches. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-7.html#support-for-import-d-from-cjs-from-commonjs-modules-with---esmoduleinterop

## 2. wasm files

`dumber` wraps `.wasm` files into AMD modules. You can just use `import foo from './foo.wasm';` in your code.

To put wasm inside text format bundle file, wasm is encoded in base64 string (which means the size is ineffectively larger than the original binary size).

See an example in [https://github.com/dumberjs/examples/tree/master/aurelia-esnext-scss-jasmine](https://github.com/dumberjs/examples/tree/master/aurelia-esnext-scss-jasmine).

> Same as dealing with babel and TypeScript, `dumber` doesn't care how you compile the original source (like c or c++ source file) into wasm file. It's user's responsibility to handle the compilation (from c or other languages to wasm) in gulp pipeline. On top of that, user also needs `import foo from './foo.wasm';` instead of `import foo from './foo.c';`, because `dumber` has no idea that the final `foo.wasm` was compiled from `foo.c`.

## 3. json files

`dumber` wraps `.json` files into AMD modules.

For Nodejs compatibility, json modules are following Nodejs sematics, the result is something like `module.exports = { ... };`. Babel has no problem to understand it, but TypeScript users has to pick one of following two ways to support it.

1. Turn on `esModuleInterop`. https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-7.html#support-for-import-d-from-cjs-from-commonjs-modules-with---esmoduleinterop
2. Or use ESM (esnext) as the output module format (`"module": "esnext"` in `tsconfig.json`). With ESM as resulting format, `dumber` can normalise the import statements same as Babel.

## 4. css files

`dumber` wraps `.css` files into simple AMD text modules. By default, `dumber` also ships a css extension plugin for [`dumber-module-loader`](https://github.com/dumberjs/dumber-module-loader) to inject the css into HTML head when you import the css file. User can opt-out the default behaviour with `injectCss: false`.

```js
const dr = dumber({
  injectCss: false,
  // ...
});
```

> Similar to js files, `dumber` doesn't understand `.scss` or `.less` files. You need to use gulp-sass or gulp-less to transpile those files into `.css` vinyl files before sending to `dumber`.

If your source file is `foo.scss`, you can use either `import './foo.css';` or `import './foo.scss';`. Recommend you to use `import './foo.scss';`, as it's least surprising, and compatible with test frameworks Jest and AVA running in Nodejs environment.

The real bundled module id is `foo.css` as it is the only file `dumber` saw (after gulp-sass processed it). [`dumber-module-loader`](https://github.com/dumberjs/dumber-module-loader) actually resolves module id `foo.scss` to `foo.css` at runtime.

## 5. all other files

All unknown files are treated as text modules. The includes html files.

> This means `dumber` cannot bundle binary files like jpeg pictures. (`wasm` is the only binary format `dumber` accepts.)

## 6. npm files

npm files are not explicitly added to gulp stream. `dumber` automatically brings in the needed npm files based on code tracing.

## About gulp stream

In the gulp code sample, [merge2](https://github.com/teambition/merge2) is used to merge multiple gulp streams. Note gulp actually supports adding streams to pipeline, it was preprocessors (babel and sass) we need to isolate. Following code sample uses [gulp-if](https://github.com/robrich/gulp-if) instead of merge2 to do the same thing.

```js
const dumber = require('gulp-dumber');
const dr = dumber({/* ... */});

function build() {
  return gulp.src('src/**/*.json')
    .pipe(gulp.src('src/**/*.html'))
    .pipe(gulp.src('src/**/*.js'))
    .pipe(gulpIf(file => file.extname === '.js', babel()))
    .pipe(gulp.src('src/**/*.scss'))
    .pipe(gulpIf(file => file.extname === '.scss', sass()))
    // Here is dumber doing humble bundling.
    // The extra call `dr()` is designed to cater watch mode.
    .pipe(dr())
    .pipe(gulp.dest('dist'));
}
```

Or simply:

```js
const dumber = require('gulp-dumber');
const dr = dumber({/* ... */});

function build() {
  return gulp.src('src/**/*.{json,html,js,scss}')
    .pipe(gulpIf(file => file.extname === '.js', babel()))
    .pipe(gulpIf(file => file.extname === '.scss', sass()))
    // Here is dumber doing humble bundling.
    // The extra call `dr()` is designed to cater watch mode.
    .pipe(dr())
    .pipe(gulp.dest('dist'));
}
```

## Module id

`dumber` assigns a module id for every resource (file) it bundled.

## Module id for local source files

Module id for a local source is relative to [src path](./options/src) (default to `"src"`).
* For local src file `src/foo/bar.js`, the module id is `foo/bar.js`.
* For local src file `src/foo/bar.css` (or any other non-js file), the module id is `foo/bar.css`.

> Early versions of `dumber` normalises module id for JavaScript files like `src/foo/bar.js` to `foo/bar` (stripped `.js` file extension), because that's usually how user requires or imports it. Latest version of `dumber` changed the behaviour to retain all file extension in module id. This is for better support of latest Nodejs ES Modules `.mjs` and `.cjs` files, because Nodejs now requires [mandatory file extensions](https://nodejs.org/dist/latest-v12.x/docs/api/esm.html#esm_mandatory_file_extensions).

> No matter how `dumber` assigns module id internally, `bar.js` or `bar`, it doesn't affect user code. User can import either `./bar.js` or `./bar`. `dumber` allows user code to omit `.js`, `.cjs` or `.mjs` file extension. This module id compatibility is handled by [`dumber-module-loader`](https://github.com/dumberjs/dumber-module-loader).

## Module id for npm package file

Module id for a npm package file starts with npm package name.

* For npm package file `node_modules/foo/bar.js`, the module id is `foo/bar.js`.
* For scoped npm package file `node_modules/@scoped/foo/bar.js`, the module id is `@scoped/foo/bar.js`.

## Above-surface module id

When a local file is out of [src path](./options/src), for example `foo/bar.js` in folder `foo/`, not folder `src/`, the module id assigned will be `../foo/bar.js` as if it's relative to `src/` folder.

RequireJS doesn't support absolute module id starting with `../` (it confuses RequireJS as relative module id).

[`dumber-module-loader`](https://github.com/dumberjs/dumber-module-loader) supports absolute module id starting with `../`. This kind of module id is called "above-surface" module id. It's designed to allow dumber to flexibly import files out of [src path](./options/src).

## Two module spaces

[`dumber-module-loader`](https://github.com/dumberjs/dumber-module-loader) separates local sources and npm packages into two module spaces: `user` and `package`. This is designed to solve one RequireJS problem: local module id conflicts with npm package names.

The module spaces are totally transparent to the users of `dumber` bundler. If you are interested, read more at [https://github.com/dumberjs/dumber-module-loader](https://github.com/dumberjs/dumber-module-loader).
