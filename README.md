IOpipe
---------------------------------------
Apache 2.0 licensed.

IOpipe is a toolkit for building event-driven applications
running locally or in the cloud (AWS Lambda, Google Cloud Functions,
Azure Functions). At its core, IOpipe provides a flow-based model for
building functional, scalable applications through the chaining
of single-function modules.

Kernels take and transform input and communicate over the networking,
operating in a fashion to Unix pipes. A kernel may receive input or send output to/from
web service requests, functions, or local applications.

![Build Status](https://circleci.com/gh/iopipe/iopipe.png?circle-token=eae431abda6b19dbfca597af818bb01092211272)
[![Coverage Status](https://coveralls.io/repos/github/iopipe/iopipe/badge.svg?branch=master&t=UYi1cn)](https://coveralls.io/github/iopipe/iopipe?branch=master)

---------------------------------------
Usage
---------------------------------------

### NodeJS SDK:

The NodeJS SDK provides a generic callback chaining mechanism which allows
mixing HTTP(S) requests/POSTs, function calls, and kernels. Callbacks
receive the return of the previous function call or HTTP body.

The callback variable received by a function is *also* an AWS Lambda-compatible
"context" object. Because of this, you can chain standard callback-based NodeJS
functions, and functions written for AWS Lambda.

```javascript
var iopipe = require("iopipe")()

/* Get HTTP data, process it with SomeScript, and POST the results.
   Note that com.example.SomeScript would be present in .iopipe/filter_cache/ */
iopipe.exec("http://localhost/get-request",
            "com.example.SomeScript",
            "http://otherhost.post")

// Users may chain functions and HTTP requests.
iopipe.exec(function(_, callback) { callback("something") },
            function(arg, callback) { callback(arg) },
            "http://otherhost.post",
            your_callback)

// A function may also be returned then executed later.
var f = iopipe.define("http://fetch", "https://post")
f()

// A defined function also accepts parameters
var echo = require("iopipe-echo")
var f = iopipe.define(echo, console.log)
f("hello world")

/* Create an AWS Lambda function from any NodeJS function /w callback.
   The callback becomes the equivilent of a done or success call on AWS. */
export.handler = iopipe.define(function(event, callback) {
  console.log(event)
  callback()
})

/* Of course, this method chaining also works for creating AWS Lambda code.
   This example will fetch HTTP data from the URL in the event's 'url' key
   and return a SHA-256 of the retrieved content. */
var crypto = require("crypto")
export.handler = iopipe.define(iopipe.property("url"),
                               iopipe.fetch,
                               (event, callback) => {
                                  callback(crypto
                                           .createHash('sha256')
                                           .update(event)
                                           .digest('hex'))
                               })
```

For more information on using the NodeJS SDK, please refer to its documentation:
***https://github.com/iopipe/iopipe/blob/master/docs/nodejs.md***

---------------------------------------
Kernel functions
---------------------------------------

Requests and responses are translated using kernels, and
may pipe to other kernels, or to/from web service endpoints.

Kernels simply receive request or response data and output
translated request or response data.

Example:

```javascript
module.exports = function(input, context) {
  context.done("I'm doing something with input: {0}".format(input))
}
```

Functions should expect a "context" parameter which may be called
directly as a callback, but also offers the methods 'done', 'success',
and 'fail'. Users needing, for any reason, to create a context manually
may call iopipe.create_context(callback).

For more on writing filters see:
***https://github.com/iopipe/iopipe/blob/master/docs/kernels.md***

### CLI

A Go-based CLI exists to create and export npm modules, share code,
and provide runtime of magnetic kernels.

Download the [latest binary release](https://github.com/iopipe/iopipe/releases) and chmod 755 the file.

Building from source? See [Build & Install from source](#build--install-from-source).

Alternatively, download & alias our Docker image:

```bash
$ docker pull iopipe/iopipe:trunk
$ docker run --name iopipe-data iopipe/iopipe:trunk
$ eval $(echo "alias iopipe='docker run --rm --volumes-from iopipe-data iopipe/iopipe:trunk'" | tee -a ~/.bashrc)
$ iopipe --help
```

OS-specific packages are forthcoming.

### Command-line Examples

```sh
# Import a kernel and name it com.example.SomeScript
$ iopipe import --name com.example.SomeScript - <<<'input'

# List kernels
$ iopipe list

# Fetch response and process it with com.example.SomeScript
$ iopipe --debug exec http://localhost/some-request com.example.SomeScript

# Fetch response and convert it with SomeScript, sending the result to otherhost
$ iopipe --debug exec http://localhost/some-request com.example.SomeScript \
                      http://otherhost/request

# Fetch response and convert it with SomeScript, send that result to otherhost,
# & converting the response with the script ResponseScript
$ iopipe --debug exec http://localhost/some-request com.example.SomeScript \
                      http://otherhost/request some.example.ResponseScript

# Export an NPM module:
$ iopipe export --name my-module-name http://localhost/some-request com.example.SomeScript
```

---------------------------------------
Build & Install from source
---------------------------------------

With a functioning golang 1.5 development environment:

```bash
$ go build
$ ./iopipe --help
```

Alternatively use Docker to build & deploy:

```bash
$ docker build -t iopipe-dev .
$ docker run --name iopipe-data iopipe-dev
$ eval $(echo "alias iopipe='docker run --rm --volumes-from iopipe-data iopipe-dev'" | tee -a ~/.bashrc)
$ iopipe --help
```
---------------------------------------
Security
---------------------------------------

Kernels are executed in individual virtual machines
whenever allowed by the executing environment.
The definition of a virtual machine here is lax,
such that it may describe a Javascript VM,
a Linux container, or a hardware-assisted x86
virtual machine. Users should exercise caution
when running community created kernels.

It is a project priority to make fetching, publishing,
and execution of kernels secure for a
production-ready 1.0.0 release.

Modules are fetched and stored using sha256 hashes,
providing an advantage over module-hosting mechanisms
which are based simply on a name and version. Future
versions of IOpipe will likely implement TUF for
state-of-the-art software assurance.

Contact security@iopipe.com for questions.

---------------------------------------
LICENSE
---------------------------------------

Apache 2.0
