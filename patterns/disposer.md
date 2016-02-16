# The Promise Disposer Pattern

Often we have to perform resource management in code. For example - release a DB connection:


```js
var users = getDb().then(conn => 
    conn.query("SELECT name FROM users").then(users => {
      conn.release();
      return users;
    }, e => {
      conn.release();
      throw e;
    })
});
```

The problem with the above approach is that if you forget releasing the connection after every single
time you perform `getDb` you have a resource leak that might freeze your app eventually
when it runs out of the resource you're leaking.

The above approach means that the user has to manually manage resources. 

The user might in one place do:

```js
var users = getDb().then(function(conn) {
     return conn.query("SELECT name FROM users");
});
```

Which will leak a database connection that was never closed. 

The disposer pattern is a way to couple a scope of code with owning the resource. By binding the resource
to a scope we make sure it is always released when we're done with it and we can't easily forget 
to release it. It is similar to `using` in C#, `with` in Python and try-with-resource in Java 
as well as RAII in C++.


It looks like:

```js
 withResource(function(resource) {
     return fnThatDoesWorkWithResource(resource); // returns a promise
 }).then(function(result) {
    // resource disposed here
 });
```

If we wrote our code as:

```JS
function withDb(work) {
    var _db;
    return myDbDriver.getConnection().then(function(db) {
      _db = db; // keep reference 
      return work(db); // perform work on db
    }).then(v => {
      if (_db) {
        _db.release();
      }
      return v;
    }, e => {
      if(_db) {
        _db.release();
      }
      throw e;
    });
}
```

We could write our above code as:

```js
withDb(conn => 
  conn.query("SELECT name FROM users");
).then(users => {
   // connection released here
});
```

Userland examples of users of the disposer pattern are [sequelize](https://github.com/sequelize/sequelize/issues/1939)
and [knex](https://github.com/tgriesser/knex/issues/265) (bookshelf's query builder). It's also possible to use it
for simpler things like hiding a loader when all AJAX requests completed for instance. 
