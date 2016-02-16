##The Explicit Construction Anti-Pattern

This is the most common anti-pattern. It is easy to fall into this when you haven't really grasped
promises. It's also sometimes called the promise constructor anti-pattern. Let's recap: promises 
are about making asynchronous code retain most of the lost properties of synchronous code such 
as flat indentation and one exception channel.

This pattern is also called the deferred anti-pattern.

In the explicit construction anti-pattern, promise objects are created for no reason, complicating code.

First example is creating a new promise object when you already have a promise or thenable:

```js
function getFullName(userId) {
    return new Promise(function(resolve, reject) {
      db.getUsers(userId).then(data => {
        resolve(data.fullName);
      });
    });
}
```

This superfluous wrapping is also dangerous, any kind of errors and rejections are swallowed and
not propagated to the caller of this function.

Instead of using the above the code should simply return the promise it already has and propagate
values using `return`:

```js
function getFullName(userId) {
    return db.getUsers(userId).then(data => 
        data.fullName)
      );
    });
}
```

Not only is the code shorter but more importantly, if there is any error it will propagate properly to 
the final consumer.

**So when should the promise constructor be used?**

Well simply, when you have to.

You might have to use explicit wrapping object when wrapping a callback API.  
