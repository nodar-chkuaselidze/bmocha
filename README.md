# bmocha

Alternative implementation of [Mocha][mocha] (requires no external dependencies
for security purposes).

## Usage

Bmocha's CLI mimics Mocha's CLI for most features:

```
$ bmocha --help

  Usage: bmocha [options] [files]

  Options:

    -V, --version           output the version number
    -c, --colors            force enabling of colors
    -C, --no-colors         force disabling of colors
    -O, --reporter-options  reporter-specific options
    -R, --reporter <name>   specify the reporter to use (default: spec)
    -S, --sort              sort test files
    -b, --bail              bail after first test failure
    -g, --grep <pattern>    only run tests matching <pattern>
    -f, --fgrep <string>    only run tests containing <string>
    -i, --invert            inverts --grep and --fgrep matches
    -r, --require <name>    require the given module
    -s, --slow <ms>         "slow" test threshold in milliseconds [75]
    -t, --timeout <ms>      set test-case timeout in milliseconds [2000]
    -u, --ui <name>         specify user-interface (bdd) (default: bdd)
    --interfaces            display available interfaces
    --exit                  force shutdown of the event loop after test run
    --allow-uncaught        enable uncaught errors to propagate
    --no-timeouts           disables timeouts
    --recursive             include sub directories
    --reporters             display available reporters
    --retries <times>       set numbers of time to retry a failed test case
    --file <file>           include a file to be ran during the suite
    --exclude <file>        a file to ignore
    -l, --listen            serve client-side test files (requires browserify)
    -p, --port <port>       port to listen on [8080]
    -o, --open              open browser after serving
    -H, --headless          run tests in headless chrome
    --chrome <path>         chrome binary to use for headless mode
    -m, --cmd <cmd>         set browser command
    -z, --console           use console in browser
    -h, --help              output usage information
```

### Example

``` bash
$ bmocha --reporter spec test.js
```

## Docs

Because bmocha is more or less a full clone of mocha, the MochaJS docs should
be sufficient for any typical use-case. See [mochajs.org][mocha].

## Features

### Easily Auditable Code (the "why?")

There have been a number of NPM package attacks in the past. The most recent
being an attack on the popular `event-stream` library. There are many projects
with _financial_ components to them, cryptocurrency projects in particular.

Mocha pulls in a number of dependencies (23 with dedupes, and an even greater
amount of dev dependencies):

```
$ npm ls
mocha@5.2.0
├── browser-stdout@1.3.1
├── commander@2.15.1
├─┬ debug@3.1.0
│ └── ms@2.0.0
├── diff@3.5.0
├── escape-string-regexp@1.0.5
├─┬ glob@7.1.2
│ ├── fs.realpath@1.0.0
│ ├─┬ inflight@1.0.6
│ │ ├── once@1.4.0
│ │ └── wrappy@1.0.2
│ ├── inherits@2.0.3
│ ├── minimatch@3.0.4
│ ├─┬ once@1.4.0
│ │ └── wrappy@1.0.2
│ └── path-is-absolute@1.0.1
├── growl@1.10.5
├── he@1.1.1
├─┬ minimatch@3.0.4
│ └─┬ brace-expansion@1.1.11
│   ├── balanced-match@1.0.0
│   └── concat-map@0.0.1
├─┬ mkdirp@0.5.1
│ └── minimist@0.0.8
└─┬ supports-color@5.4.0
  └── has-flag@3.0.0
```

As maintainers of several cryptocurrency projects, we find this attack surface
to be far too large for comfort. Although we of course trust the mocha
developers, only one of its dependencies need be compromised in order to
potentially steal bitcoin or API keys.

As a result, bmocha pulls in _zero_ dependencies: what you see is what you get.
The code is a couple thousand lines, residing in `lib/` and `bin/`.

### Headless Chrome & Browser Support

If browserify is installed as a global or peer dependency, running tests in
headless chrome is as easy as:

``` bash
$ bmocha -H test.js
```

Chromium or chrome must be installed in one of the usual locations depending on
your OS. If both are installed, bmocha prefers chromium over chrome.

The tests will run in a browserify environment with some extra features:

- `console.{log,error,info,warn,dir}` and `process.{stdout,stderr}` will work
  as expected.
- `process.{exit,abort}` will work as expected.
- The `fs` module will work in "read-only" mode. All of the read calls,
  including `access`, `exists`, `stat`, `readdir`, and `readFile` will all work
  properly (sync and async). As a security measure, they will only be able to
  access your current working directory and nothing else (note that this is not
  a jail or chroot: if you have a symlink out of your module's directory, the
  tests will be able to access this).

If your chrome binary is somewhere non-standard, you are able to pass the
`--chrome` flag.

``` bash
$ bmocha --chrome="$(which google-chrome-unstable)" test.js
```

To run the tests in your default non-headless browser:

``` bash
$ bmocha -o test.js
```

Will open a browser window and display output in the DOM.

To run with the output written to the console instead:

``` bash
$ bmocha -oz test.js
```

To pass a custom browser to open, use `-m` instead of `-o`:

``` bash
$ bmocha -m 'chromium %s' test.js
```

Where `%s` is where you want the server's URL to be placed.

For example, to run chromium in app mode:

``` bash
$ bmocha -m 'chromium --app=%s' test.js
```

By default, bmocha will start an HTTP server listening on a random port. To
specify the port:

``` bash
$ bmocha -p 8080 -m 'chromium --app=%s' test.js
```

And finally, to simply start an http server without any browser action, the
`-l` flag is available:

``` bash
$ bmocha -lp 8080 test.js
```

#### Support for Workers

In the browser, your code may be using workers. To notify bmocha of this, a
global `register` call is exposed during test execution.

``` js
function createWorker() {
  if (process.env.BMOCHA) {
    // Usage: register([desired-url-path], [filesystem-path]);
    register('/worker.js', [__dirname, 'worker.js']);
  }

  return new Worker('/worker.js');
}
```

When `createWorker` is called, the bmocha server is notified that it should
compile and serve `${__dirname}/worker.js` as `/worker.js`.

### Arrow Functions

Bmocha supports arrow functions in a backwardly compatible way:

``` js
describe('Suite', (ctx) => {
  ctx.timeout(1000);

  it('should skip test', (ctx) => {
    ctx.skip();
    assert(1 === 0);
  });
});
```

For `it` calls, the argument name _must_ be `ctx`. Any single parameter
function with a `ctx` argument is considered a "context" variable instead of a
callback.

However, it is also possible to use callbacks _and_ contexts.

``` js
describe('Suite', (ctx) => {
  ctx.timeout(200);

  it('should run test (async)', (done) => {
    done.slow(10);
    assert(typeof done === 'function');
    setTimeout(done, 20);
  });
});
```

The context's properties are injected into the callback whenever a callback
function is requested.

#### Single-Letter Context Variables

Typing `ctx` repeatedly may seem unwieldly compared to writing normal arrow
functions. For this reason, there are 3 more "reserved arguments": `$`, `_`,
and `x`.

``` js
describe('Suite', () => {
  it('should skip test', $ => $.skip());
  it('should skip test', _ => _.skip());
  it('should skip test', x => x.skip());
});
```

All three will work as "context variables".

### Fixes for Mocha legacy behavior

Since we're building from scratch with zero dependents, we have an opportunity
to fix some of the bugs in Mocha.

For example:

``` js
describe('Suite', () => {
  it('should fail', (cb) => {
    cb();
    throw new Error('foobar');
  });
});
```

```
$ mocha test.js


  Suite
    ✓ should fail


  1 passing (6ms)
```

The above passes in mocha and _swallows_ the error. We don't want to interfere
with existing mocha tests, but we can output a warning to the programmer:

```
$ bmocha test.js

  Suite
    ✓ should fail
      ! swallowed error as per mocha behavior:
        Error: foobar

  1 passing (4ms)
```

Likewise, the following tests also pass in mocha without issue:

``` js
describe('Suite', () => {
  it('should fail (unhandled rejection)', () => {
    new Promise((resolve, reject) => {
      reject(new Error('foobar'));
    });
  });

  it('should fail (resolve & resolve)', () => {
    return new Promise((resolve, reject) => {
      resolve(1);
      resolve(2);
    });
  });

  it('should fail (resolve & reject)', () => {
    return new Promise((resolve, reject) => {
      resolve(3);
      reject(new Error('foobar'));
    });
  });

  it('should fail (resolve & throw)', () => {
    return new Promise((resolve, reject) => {
      resolve(4);
      throw new Error('foobar');
    });
  });
});
```

```
$ mocha test.js


  Suite
    ✓ should fail (unhandled rejection)
    ✓ should fail (resolve & resolve)
    ✓ should fail (resolve & reject)
    ✓ should fail (resolve & throw)


  4 passing (7ms)
```

Bmocha will report and catch unhandled rejections, multiple resolutions, along
with other strange situations:

```
$ bmocha test.js

  Suite
    1) should fail (unhandled rejection)
    2) should fail (resolve & resolve)
    3) should fail (resolve & reject)
    4) should fail (resolve & throw)

  0 passing (4ms)
  4 failing

  1) Suite
       should fail (unhandled rejection):

      Unhandled Error: foobar

      reject(new Error('foobar'));
             ^

      at Promise (/home/bmocha/test.js:4:14)
      at new Promise (<anonymous>)
      at Context.it (/home/bmocha/test.js:3:5)

  2) Suite
       should fail (resolve & resolve):

      Uncaught Error: Multiple resolves detected for number.

      2

  3) Suite
       should fail (resolve & reject):

      Uncaught Error: Multiple rejects detected for error.

      Error: foobar
          at Promise (/home/bmocha/test.js:18:14)
          at new Promise (<anonymous>)
          at Context.it (/home/bmocha/test.js:16:12)

      reject(new Error('foobar'));
             ^

  4) Suite
       should fail (resolve & throw):

      Uncaught Error: Multiple rejects detected for error.

      Error: foobar
          at Promise (/home/bmocha/test.js:25:13)
          at new Promise (<anonymous>)
          at Context.it (/home/bmocha/test.js:23:12)

      throw new Error('foobar');
            ^
```

Mocha tends to die in very strange ways on uncaught errors. Take for instance:

``` js
describe('Suite', () => {
  it('should fail (setImmediate)', () => {
    setImmediate(() => {
      throw new Error('foobar 1');
    });
  });

  it('should not fail (setTimeout)', () => {
    setTimeout(() => {
      throw new Error('foobar 2');
    }, 1);
  });
});
```

```
$ mocha test.js


  Suite
    ✓ should fail (setImmediate)
    1) should fail (setImmediate)

  1 passing (5ms)
  1 failing

  1) Suite
       should fail (setImmediate):
     Uncaught Error: foobar 1
      at Immediate.setImmediate (test.js:4:13)



    ✓ should not fail (setTimeout)
```

The garbled output shown above is very confusing and not very user friendly.

In bmocha, the results are as such:

```
$ bmocha test.js

  Suite
    1) should fail (setImmediate)
    ✓ should not fail (setTimeout)

  1 passing (5ms)
  1 failing

  1) Suite
       should fail (setImmediate):

      Uncaught Error: foobar 1

      throw new Error('foobar 1');
            ^

      at Immediate.setImmediate (/home/bmocha/test.js:4:13)
      at processImmediate (timers.js:632:19)


  An error occurred outside of the test suite:

    Uncaught Error: foobar 2

    throw new Error('foobar 2');
          ^

    at Timeout.setTimeout [as _onTimeout] (/home/bmocha/test.js:10:13)
    at listOnTimeout (timers.js:324:15)
    at processTimers (timers.js:268:5)
```

A note on uncaught errors, unhandled rejections, and multiple resolutions:
Mocha does not even handle the latter, but it tends to die strangely on the
former two. In fact, it dies almost instantly. Bmocha will attempt to "attach"
uncaught errors to the currently running test and reject it. If there is no
currently running test, bmocha will buffer the error until the end, at which
point it will list all of the uncaught errors. If bmocha is no longer running
at all, the error will be output and the process will be exited.

This can lead to differing output on each run if your process has uncaught
errors. Running it again, bmocha was able to attach the error to the currently
running test:

```
$ bmocha test.js

  Suite
    1) should fail (setImmediate)
    2) should not fail (setTimeout)

  0 passing (4ms)
  2 failing

  1) Suite
       should fail (setImmediate):

      Uncaught Error: foobar 1

      throw new Error('foobar 1');
            ^

      at Immediate.setImmediate (/home/bmocha/test.js:4:13)
      at processImmediate (timers.js:632:19)

  2) Suite
       should not fail (setTimeout):

      Uncaught Error: foobar 2

      throw new Error('foobar 2');
            ^

      at Timeout.setTimeout [as _onTimeout] (/home/bmocha/test.js:10:13)
      at listOnTimeout (timers.js:324:15)
      at processTimers (timers.js:268:5)
```

Mocha also only _warns_ when _explicitly_ passed a non-existent test. This is a
shortcoming in CI situations which may only look at the exit code.

```
$ mocha test.js non-existent.js || echo 1
Warning: Could not find any test files matching pattern: non-existent.js


  Suite
    ✓ should pass


  1 passing (3ms)
```

Bmocha will fail outright:

```
$ bmocha test.js non-existent.js || echo 1
File not found: non-existent.js.
1
```

## Raw API

To explicitly run bmocha as a module:

### JS

Mocha accepts a `Stream` object for output.

``` js
const assert = require('assert');
const {Mocha} = require('bmocha');

const mocha = new Mocha({
  stream: process.stdout,
  reporter: 'nyan',
  fgrep: 'Foobar'
});

const code = await mocha.run(() => {
  describe('Foobar', function() {
    this.timeout(5000);

    it('should check 1 == 1', function() {
      this.retries(10);
      assert.equal(1, 1);
    });
  });
});

if (code !== 0)
  process.exit(code);
```

### Browser

Running in the browser is similar. To output to the DOM, a `DOMStream` object
is available:

``` js
const {Mocha, DOMStream} = require('bmocha');
const stream = new DOMStream(document.body);
const mocha = new Mocha(stream);

await mocha.run(...);
```

Likewise, a `ConsoleStream` object is available to output to the console:

``` js
const {Mocha, ConsoleStream} = require('bmocha');
const stream = new ConsoleStream(console);
const mocha = new Mocha(stream);

await mocha.run(...);
```

## Contribution and License Agreement

If you contribute code to this project, you are implicitly allowing your code
to be distributed under the MIT license. You are also implicitly verifying that
all code is your original work. `</legalese>`

## License

- Copyright (c) 2018, Christopher Jeffrey (MIT License).

See LICENSE for more info.

[mocha]: https://mochajs.org/
