# RSVP.js  [![Build Status](https://secure.travis-ci.org/tildeio/rsvp.js.png?branch=master)](http://travis-ci.org/tildeio/rsvp.js)

RSVP.js provides simple tools for organizing asynchronous code.

Specifically, it is a tiny implementation of Promises/A+ and a
mixin for turning objects into event targets.

It works in node and the browser.

## downloads

- [rsvp-latest](http://rsvpjs-builds.s3.amazonaws.com/rsvp-latest.js)
- [rsvp-latest (amd)](http://rsvpjs-builds.s3.amazonaws.com/rsvp-latest.amd.js)

## Promises

## Bower

`bower install -S rsvp`

## NPM

`npm install --save rsvp`

`RSVP.Promise` is an implementation of
[Promises/A+](http://promises-aplus.github.com/promises-spec/) that passes the
[test suite](https://github.com/promises-aplus/promises-tests).

It delivers all promises asynchronously, even if the value is already
available, to help you write consistent code that doesn't change if the
underlying data provider changes from synchronous to asynchronous.

It is compatible with [TaskJS](http://taskjs.org/), a library by Dave
Herman of Mozilla that uses ES6 generators to allow you to write
synchronous code with promises. It currently works in Firefox, and will
work in any browser that adds support for ES6 generators. See the
section below on TaskJS for more information.

### Basic Usage

```javascript
var promise = new RSVP.Promise(function(resolve, reject){
  // succeed
  resolve(value);
  // or reject
  reject(error);
});

promise.then(function(value) {
  // success
}, function(value) {
  // failure
});
```

Once a promise has been resolved or rejected, it cannot be resolved or
rejected again.

Here is an example of a simple XHR2 wrapper written using RSVP.js:

```javascript
var getJSON = function(url) {
  var promise = new RSVP.Promise(function(resolve, reject){
    var client = new XMLHttpRequest();
    client.open("GET", url);
    client.onreadystatechange = handler;
    client.responseType = "json";
    client.setRequestHeader("Accept", "application/json");
    client.send();

    function handler() {
      if (this.readyState === this.DONE) {
        if (this.status === 200) { resolve(this.response); }
        else { reject(this); }
      }
    };
  });

  return promise;
};

getJSON("/posts.json").then(function(json) {
  // continue
}, function(error) {
  // handle errors
});
```

### Chaining

One of the really awesome features of Promises/A+ promises are that they
can be chained together. In other words, the return value of the first
resolve handler will be passed to the second resolve handler.

If you return a regular value, it will be passed, as is, to the next
handler.

```javascript
getJSON("/posts.json").then(function(json) {
  return json.post;
}).then(function(post) {
  // proceed
});
```

The really awesome part comes when you return a promise from the first
handler:

```javascript
getJSON("/post/1.json").then(function(post) {
  // save off post
  return getJSON(post.commentURL);
}).then(function(comments) {
  // proceed with access to posts and comments
});
```

This allows you to flatten out nested callbacks, and is the main feature
of promises that prevents "rightward drift" in programs with a lot of
asynchronous code.

Errors also propagate:

```javascript
getJSON("/posts.json").then(function(posts) {

}).catch(function(error) {
  // since no rejection handler was passed to the
  // first `.then`, the error propagates.
});
```

You can use this to emulate `try/catch` logic in synchronous code.
Simply chain as many resolve callbacks as a you want, and add a failure
handler at the end to catch errors.

```javascript
getJSON("/post/1.json").then(function(post) {
  return getJSON(post.commentURL);
}).then(function(comments) {
  // proceed with access to posts and comments
}).catch(function(error) {
  // handle errors in either of the two requests
});
```

You can also use `catch` for error handling, which is a shortcut for
`then(null, rejection)`, like so:

```javascript
getJSON("/post/1.json").then(function(post) {
  return getJSON(post.commentURL);
}).catch(function(error) {
  // handle errors
});
```

## Error Handling

There are times when dealing with promises that it seems like any errors
are being 'swallowed', and not properly raised. This makes is extremely
difficult to track down where a given issue is coming from. Thankfully,
`RSVP` has a solution for this problem built in.

You can register functions to be called when an uncaught error occurs
within your promises. These callback functions can be anything, but a common
practice is to call `console.assert` to dump the error to the console.

```javascript
RSVP.on('error', function(event) {
  console.assert(false, event.detail);
});
```

**NOTE:** Usage of `RSVP.configure('onerror', yourCustomFunction);` is
deprecated in favor of using `RSVP.on`.

## Arrays of promises

Sometimes you might want to work with many promises at once. If you
pass an array of promises to the `all()` method it will return a new
promise that will be fulfilled when all of the promises in the array
have been fulfilled; or rejected immediately if any promise in the array
is rejected.

```javascript
var promises = [2, 3, 5, 7, 11, 13].map(function(id){
  return getJSON("/post/" + id + ".json";
});

RSVP.all(promises).then(function(posts) {
  // posts contains an array of results for the given promises
}).catch(function(reason){
  // if any of the promises fails.
});
```

## Hash of promises

If you need to reference many promises at once (like `all()`), but would like
to avoid encoding the actual promise order you can use `hash()`. If you pass
an object literal (where the values are promises) to the `hash()` method it will
return a new promise that will be fulfilled when all of the promises have been
fulfilled; or rejected immediately if any promise is rejected.

The key difference to the `all()` function is that both the fulfillment value
and the argument to the `hash()` function are object literals. This allows
you to simply reference the results directly off the returned object without
having to remember the initial order like you would with `all()`.

```javascript
var promises = {
  posts: getJSON("/posts.json"),
  users: getJSON("/users.json")
};

RSVP.hash(promises).then(function(results) {
  console.log(results.users) // print the users.json results
  console.log(results.posts) // print the posts.json results
});
```

## All settled and hash settled

Sometimes you want to work with several promises at once, but instead of
rejecting immediately if any promise is rejected, as with `all()` or `hash()`,
you want to be able to inspect the results of all your promises, whether
they fulfill or reject. For this purpose, you can use `allSettled()` and
`hashSettled()`. These work exactly like `all()` and `hash()`, except that
they fulfill with an array or hash (respectively) of the constituent promises'
result states. Each state object will either indicate fulfillment or
rejection, and provide the corresponding value or reason. The states will take
one of the following formats:

```javascript
{ state: 'fulfilled', value: value }
  or
{ state: 'rejected', reason: reason }
```

## Deferred

RSVP also has a RSVP.defer() method that returns a deferred object of the form 
`{ promise, resolve(x), reject(r) }`. This creates a deferred object without 
specifying how it will be resolved. However, the `RSVP.Promise` constructor is 
generally a better and less error-prone choice; we recommend using it in 
preference to `RSVP.defer()`.


## TaskJS

The [TaskJS](http://taskjs.org/) library makes it possible to take
promises-oriented code and make it synchronous using ES6 generators.

Let's review an earlier example:

```javascript
getJSON("/post/1.json").then(function(post) {
  return getJSON(post.commentURL);
}).then(function(comments) {
  // proceed with access to posts and comments
}).catch(function(reason) {
  // handle errors in either of the two requests
});
```

Without any changes to the implementation of `getJSON`, you could write
the following code with TaskJS:

```javascript
spawn(function *() {
  try {
    var post = yield getJSON("/post/1.json");
    var comments = yield getJSON(post.commentURL);
  } catch(error) {
    // handle errors
  }
});
```

In the above example, `function *` is new syntax in ES6 for
[generators](http://wiki.ecmascript.org/doku.php?id=harmony:generators).
Inside a generator, `yield` pauses the generator, returning control to
the function that invoked the generator. In this case, the invoker is a
special function that understands the semantics of Promises/A, and will
automatically resume the generator as soon as the promise is resolved.

The cool thing here is the same promises that work with current
JavaScript using `.then` will work seamlessly with TaskJS once a browser
has implemented it!

## Instrumentation

```js
function listener (event) {
  event.guid      // guid of promise. Must be globally unique, not just within the implementation
  event.childGuid // child of child promise (for chained via `then`)
  event.eventName // one of ['created', 'chained', 'fulfilled', 'rejected']
  event.detail    // fulfillment value or rejection reason, if applicable
  event.label     // label passed to promise's constructor
  event.timeStamp // milliseconds elapsed since 1 January 1970 00:00:00 UTC up until now
}

RSVP.configure('instrument', true | false);
RSVP.on('created', listener);
RSVP.on('chained', listener);
RSVP.on('fulfilled', listener);
RSVP.on('rejected', listener);
```

Events are only triggered when `RSVP.configure('instrument')` is true, although
listeners can be registered at any time.

## Building & Testing

Custom tasks:

* `grunt test` - Run Mocha tests through Node and PhantomJS.
* `grunt test:phantom` - Run Mocha tests through PhantomJS (browser build).
* `grunt test:node` - Run Mocha tests through Node (CommonJS build).
* `grunt docs` - Run YUIDoc, outputting API documentation to `docs` folder.
