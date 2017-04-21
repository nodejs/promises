# Working with callbacks

This page explains how promises work with APIs that return callbacks.

When users have a callback API in a format like:

###1. One time event:

```js
process.on("rejectionHandled", function(e) { // I only ever want to deal with this once

});
```

###2. Plain non-standard callback:

```js
function request(onChangeHandler){
...
request(function(){
    // change happened
});
```

###3. Standard style callback ("nodeback"):

```js
function getStuff(dat,callback){
...
getStuff("dataParam",function(err,data){

}
```
###4. A whole library with node style callbacks:

```js
API;
API.one(function(err,data){
    API.two(function(err,data2){
        API.three(function(err,data3){

        })
    });
});
```

###How do I work with the API in promises, how do I "promisify" it?

First of all, there are plans to add support to promises in core. You can track
that here: https://github.com/nodejs/node/pull/5020

APIs should only be converted to return promises if they do not already return promises.

Promises have state, they start as pending and can settle to:

 - __fulfilled__ meaning that the computation completed successfully.
 - __rejected__ meaning that the computation failed.

Promise returning functions _should never throw_, they should return rejections instead.
Throwing from a promise returning function will force you to use both a `} catch { ` _and_ a `.catch`.
People using promisified APIs do not expect promises to throw. If you're not sure how async
APIs work in JS - please [see this answer](http://stackoverflow.com/questions/14220321) first.

###1. DOM load or other one time event:

So, creating promises generally means specifying when they settle - that means when they move
to the fulfilled or rejected phase to indicate the data is available (and can be accessed with `.then`).

```js
function load() { // our event
    return new Promise(function(resolve,reject) { // create a new promise
         process.on("rejectionHandled", resolve); // resolve it when the event happens
    });
}
```

###2. Non-Standard callback:

These APIs are rather common since well... callbacks are common in JS. In Node it is strictly prefered
that callback code you write uses the standard `(err, data)` convention.

Let's look at the common case of having `onSuccess` and `onFail`:

function getUserData(userId, onLoad, onFail){ ...
    
This would look like the following

```js
function getUserDataAsync(userId) {
    return new Promise(function(resolve,reject) {
         getUserData(userId, resolve, reject); // pass reject/resolve handlers explicitly
    });
}
```

###3. Standard style callback ("nodeback"):

Node style callbacks (nodebacks) have a particular format where the callbacks is always the last
argument and its first parameter is an error. Let's first promisify one manually:

getStuff("dataParam",function(err,data){

To:

```js
function getStuffAsync(param) { // a function that takes the parameter without the callback
    return new Promise(function(resolve,reject) { // we invoke the promise constructor
         getStuff(param,function(err,data) { // call the function internally
             if(err) return reject(err); // if it errord we reject
             resolve(data); // otherwise re resolve
         });
    });
}
```
Notes:

 - Of course, when you are in a `.then` handler you do not need to promisify things. 
  Returning a promise from a `.then` handler will resolve or reject with that promise's value. 
  Throwing from a `.then` handler is also good practice and will reject the promise. 
