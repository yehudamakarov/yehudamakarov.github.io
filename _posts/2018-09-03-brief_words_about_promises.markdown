---
layout: post
title:      "Brief Words About Promises"
date:       2018-09-03 18:35:17 -0400
permalink:  brief_words_about_promises
---


A Promise is a powerful tool. It essentially implements a placeholder for a value which will be determined later. (I may refer to it as an object but we can call it a proxy.) This is powerful because the placeholder behaves like synchronous code on the surface. We can start an asynchronous action, and before we have the final value of the operation returned to us, javascript can continue, and act seperately when we have the value. If we make a variable a Promise, the variable will immetiately be one thing, and it will change later. But we can still use that variable in the meantime. 

You can imagine that if you had console.logged the Promise immetiately after trying to do something like fetch posts, you would be able to, and you would get that Promise which is in a state called pending. It isn’t resolved yet.

If you attached something like `.then()`​ you could do something immediately when the Promise resolves, or when it is in its third state: fulfilled. If at any point during the `.then`​ chain that you set up (there could be many of them), there is an error, the Promise will become rejected: the third state it can be in. Then a `.catch()`​ method will fire no matter where in the chain control is up to. (Control is the flow of the program, where the code execution is up to.)

The Promise is an idea too. Many things implement promises. Like the library `axios`​. The idea is about chaining and making things asynchronous in an easy to read way. Check this from the MDN docs:

```js
function myAsyncFunction(url) {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open("GET", url);
    xhr.onload = () => resolve(xhr.responseText);
    xhr.onerror = () => reject(xhr.statusText);
    xhr.send();
  });
}
```

What happens when we run that function? a `new`​ Promise is made. That begins whatever it is going to do, but that Promise object already returned from the function as pending. Whereever in control the `resolve`​ function is called, the Promise object will become fulfilled! And the Promise object will pass the parameter given to `resolve`​ function over to its success handler. Above, `responseText`​ willl be the argument of the `​.then()`​ function. So we can do:

```js
// returns a promise which can call a .then() function. Right now it is 'pending'.
myAsyncFunction(url)
    // calls .then() with whatever the parameter given to resolve() was.
    .then(responseText => console.log(responseText))
    // calls .catch() with whatever the parameter given to reject was.
    .catch(statusText => console.log(statusText));
```

Let’s try this:

```js
dude = new Promise((resolve, reject) => {
	setTimeout(() => {
		console.log("HI")
		resolve("success message: HI")
	}, 2500)
})
```

Right when we execute that code, the Promise is pending. In 2.5 seconds, the Promise becomes resolved with a value of `"success message: HI”`​. It might look like this:

![Screen Shot 2018-08-31 at 18.56.18.png](https://i.imgur.com/xpDJWIc.png)

We get the return value of the variable assignment first, which is the Promise. Later, it resolves, and `​[[PromiseStatus]]`​ and `​[[PromiseValue]]`​ pop up. We also see our `console.log`.

If we call `dude.then((successMessage) => console.log(successMessage))`

![Screen Shot 2018-08-31 at 19.04.19.png](https://i.imgur.com/EKOyasv.png)

Now since each `.then()`​ will return a promise, they can be chained. Note that we already had a resolved Promise when we called `dude.then()`​. But I’ll leave you with this:

![Screen Shot 2018-08-31 at 19.07.07.png](https://i.imgur.com/nAWS3kd.png)

Check out the Mozilla docs, and then on to `async/await`.
