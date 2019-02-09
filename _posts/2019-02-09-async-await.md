---
layout: post
title:  "Await async in Javascript"
categories: [frontend]
comments: false
math: true
excerpt: "Await-async style of coding in Javascript"
---

Just as in-memory "databases" like Redis sound like a bad idea, but turn out to be really useful because we recognize that (1) we have lots of memory and (2) many applications do not need to store that much hot data anyway. It is important to realize that times are changing, constraints have to be revisited and when the right tradeoffs are made, we can have a game changer. We have to dare to think different.

Similarly, PHP, Ruby on rails, Node.js seems inefficient for server code, compared to  compiled languages like Go, but in practice, most of the workload is IO bound, and the interpreter overhead is insignificant. Node.js takes this assumption even further by using event loops and callbacks to avoid having tons of threads with huge overhead, all waiting for IO tasks to be completed.

Using only callbacks to achieve concurrency is nice, but this code style is not palatable. Search for callback hells. Since then, we have made many attempts to improve the code style. We have seen libraries like async:

```js
async.waterfall([
    function(callback) {
        callback(null, 'one', 'two');
    },
    function(arg1, arg2, callback) {
        // arg1 now equals 'one' and arg2 now equals 'two'
        callback(null, 'three');
    },
    function(arg1, callback) {
        // arg1 now equals 'three'
        callback(null, 'done');
    }
], function (err, result) {
    // result now equals 'done'
});

async.waterfall([
    myFirstFunction,
    mySecondFunction,
    myLastFunction,
], function (err, result) {
    // result now equals 'done'
});
```

This looks better but is still awkward. Ideally, the code should look as if it is synchronous. It should look as close as possible to:
```js
const x = myFirstFunction();
const y = mySecondFunction(x);
const z = myLastFunction(y);
```

A better style is with promise chaining:
```js
myFirstFunction()
.then(mySecondFunction)
.then(myLastFunction);
```

This looks slightly neater than using `async.waterfall`, but still forces you to split your code into blocks whenever you need to call an asynchronous function. Each block has to be moved into a function, anonymous or not.

To me, the current best way is with the new async-await. The code will look synchronous:
```js
const delay = (ms, result) =>
  new Promise(resolve => setTimeout(() => resolve(result), ms));

async function delays() {
  let a = await delay(800, "Hello, I'm in an");
  console.log(a);
  let b = await delay(400, "async function!");
  console.log(b);
}
delays();
```

Previously, using promise chaining, you would need something like this:
```js
delay(800, "Hello, I'm in an")
.then((a) => {
  console.log(a);
  return delay(400, "async function!");
})
.then((b) => {
  console.log(b);
});
```

The async-await style allows you to work with promises in a more natural way. Old async functions using callbacks can also be turned into functions that returns a promise via `util.promisify`.

Finally, error handling is also easier with async-await than previous approaches. We shall not go into that.
