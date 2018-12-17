---
layout: post
title:      "Fetch Vs. Axios"
date:       2018-09-03 18:35:51 -0400
permalink:  fetch_vs_axios
---


To enable your client to contact your database, you will probably end up using something like fetch, axios, or superagent. I was suggested to share why I settled on axios instead of fetch.

They are both used by the client to make asynchronous calls to a service of some sort. In a React/Redux application, it will be very handy to chain these methods using functions like `​.then()`​ and `​.catch`​ etc.

If there is a component that uses a Redux Thunk action, we can code out the main aspects of the request and response logic in the action. And if we want our component to react accordingly, we can keep its logic to itself. Consider:

```js
addPostAction({
    post: {
        title,
        description,
        tags,
        cover_image: coverImage,
        content,
    },
})
    .then(this.handleSuccess)
    .catch(this.handleError);
```

this is what `​addPostAction`​ points to:

```js
const addPostAction = newPostFromState => dispatch =>
    axios
        .post('/posts', newPostFromState)
        .then(resp => {
            dispatch({
                type: 'ADD_NEW_POST',
                payload: resp.data,
            });
        })
```

`​addPostAction`​ returns a function that takes the `dispatch`​ function as an argument. Because of this functionality, the `addPostAction` can deal with `.then`​ and such. (It is thenable.)

`​addPostAction`​ points to a function that will return a function that can use `​.then`​ or `​.catch`​. Therefore it can also use `​.then`​ or `​.catch`​.

This can be great. But the fetch code would be way more difficult to keep track of. There were two reasons for this in my experience:

1. fetch does not interpret errors from you server like 404 or 500 as errors. Read about it here: [https://developer.mozilla.org/en-US/docs/Web/API/Fetch\_API/Using\_Fetch\#Checking\_that\_the\_fetch\_was\_successful](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch#Checking_that_the_fetch_was_successful#Checking_that_the_fetch_was_successful)
2. fetch requires you to transform responses to `​json`​ before using them.

We can get this all to work with fetch! But it won't look as clean at all. We will have to remember to convert responses, and handle error flow appropriately kind of like this.

```js
fetch(url)
  .then(response => {
    return response.json().then(data => {
      if (response.ok) {
        return data;
      } else {
        return Promise.reject({status: response.status, data});
      }
    });
  })
  .then(result => console.log('success:', result))
  .catch(error => console.log('error:', error));
```

Which seemed exhausting to me compared to the above code. Hope this helps. There are lots of articles about this out there which are better than this one. But this was why I made the quick choice.
