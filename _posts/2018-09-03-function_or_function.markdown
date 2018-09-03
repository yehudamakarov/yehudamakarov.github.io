---
layout: post
title:      "`Function() {…}` Or `Function = () = {…}`"
date:       2018-09-03 22:34:36 +0000
permalink:  function_or_function
---


Sometimes you may want to answer a question accurately even though you haven’t really put words to the answer before. Check out this code:

```js
import React, { Component } from 'react';

export default class demoForFunction extends Component {
    constructor(props) {
        super(props);
        this.state = {
            count: 0,
        };
    }

    increaseCount() {
        this.setState(prevState => ({
            count: prevState.count + 1,
        }));
    }

    render() {
        const { count } = this.state;
        return (
            <div>
                <button onClick={this.increaseCount}>Count: {count} </button>
            </div>
        );
    }
}
```

This will write a button to the DOM that, when clicked, increases the value of the `​count`​ property.

I was asked why the above syntax isn’t used instead of this syntax here:

```js
increaseCount = () => {
    this.setState(prevState => ({
        count: prevState.count + 1,
    }));
}
```

as this is the one we might be used to using.

So you can answer `​this`​ will be undefined. You can also get around that like this:

```js
render() {
    const { count } = this.state;
    return <button onClick={() => { this.increaseCount() }}>Count: {count} </button>;
}
```

It would be helpful to answer WHY `​this`​ this undefined.

You’ll notice that the first piece code will throw an error when that button is pushed. It may look like this.

![Screen Shot 2018-08-29 at 19.06.59.png](https://i.imgur.com/T6NHEVL.png)

Let’s check out why. React handles DOM elements generated for us with Javascript. So when we write `<button onClick={this.increaseCount}>Count: {count} </button>`​ it may be tempting to think that the HTML element of a button has a regular old event listener attached to it. And that `​this`​ in the callback function of the event listener is undefined because is really the `window`. Being that the function is defined in strict mode, because the function is defined in a `​class`​, `​window`​ is not eligible to be the binding of `​this`​. So `​this`​ is undefined. I found out that doesn’t seem to be happening.

Let’s place a `debugger`​:

```js
increaseCount() {
    debugger;
    this.setState(prevState => ({
        count: prevState.count + 1,
    }));
}
```

And let’s run our breaking code. We will hit the debugger before the code breaks and we will be able to check out the call stack and see what some variables point to, and how React is ultimately passing around all of our code.

![Screen Shot 2018-08-29 at 19.24.42.png](https://i.imgur.com/SWt3Ehe.png)

That’s our debugger!!!

Look to the right, and you will see all the functions that were called leading up to this moment in our code. Our `increaseCount`​ function was called from inside `​callCallback`​. `​callCallback`​ was called from inside `invokeGuardedCallbackDev`​. And so on and so forth.

Let’s click on `callCallback`​ in the stack trace to see from where are function was called and how.

![Screen Shot 2018-08-29 at 19.29.18.png](https://i.imgur.com/OrzH8xM.png)

Our function is being called by `apply`​. `​apply`​ takes a parameter to explicitly bind the `​this`​ of the called function. Hmm. I wonder what React is setting the `​this`​ to in our function.

If you check the value of `​context`​, it evaluates to `​undefined`​. So React is explicitly binding `​this`​ in our function to `​undefined`​. When that happens, by the way, the default binding falls back to `​this`​. Which is the `​window`​ if we are not in strict mode, but this function is written inside of a class, so it is in strict mode. And the default binding that normally would be the `​window`​ or global object, is coerced to `​` `undefined`​.

Normally, with class functions the prototype of the object we made using the `class`​ gets a property that points to whatever `​increaseCount`​ is. This happens if it is a normal function like the one below.

```js
increaseCount() {
    this.setState(prevState => ({
        count: prevState.count + 1,
    }));
}
```

`​this`​ will be treated regularly. Eligible for binding, and susceptible to being `​undefined`​ in strict mode. But, if we use transform-class-properties that babel provides us with, (see <https://babeljs.io/docs/en/babel-plugin-transform-class-properties/>) and that should be included in `​`create-react-app​, we can have these functions always treat `​this`​ like we would expect. This is achieved by babel binding the function to the class, so that `​this`​ will always point to it.

That happens when we use:

```js
increaseCount = () => {
    this.setState(prevState => ({
        count: prevState.count + 1,
    }));
}
```

But watch out. These transformed class properties are not on the prototype. And it seems to me that every one of these functions would be a new place in memory for each object the class creates. This is actually one of the reasons we want to use prototypes in the first place.

This may actually be a good reason for some people to write:

```js
render() {
    const { count } = this.state;
    return <button onClick={() => { this.increaseCount() }}>Count: {count} </button>;
}
```

Because this way, we can define the onClick function as a function on the prototype, and get our expected results. In this case, React will call that arrow function in the onClick value with some magic I don’t understand.

![Screen Shot 2018-08-29 at 20.39.23.png](https://i.imgur.com/C4dtl2B.png)

Apparently here, our onClick function is called with the `apply`​ we saw earlier. But since we are calling an arrow function with `apply`, the passed in `​this`​ is ignored. `​this`​ will instead be implicitly bound here to the object calling the render method. That is our component. Hopefully we have an idea of how to answer the question properly next time! Babel uses `​_this`​ to transpile arrow functions and that’s why you see it in the call stack over there.

Now there is a lot more going on than what I wrote about. But it’s just a glance at how you might grab the debugger and check out some of what’s going on under the hood. To clear things up, go read <https://medium.com/dailyjs/demystifying-memory-usage-using-es6-react-classes-d9d904bc4557>. Also check out Getify (Kyle Simpson) and what he has to say about `​this`​.



