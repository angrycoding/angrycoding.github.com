# useEffect & async calls
On *sitename* there is a lot of errors / warnings, whatever you call it:

![](https://telegra.ph/file/8dcbb148cb5b45653b423.png)

This is annoying but it's more like warning and doesn't cause any major problems. At some point I was like come on, let's get rid of them. So what is causing such errors (at least in my case)? Let's look at this code:

```javascript
useEffect(() => {
  const [ something, setSomething ] = useState(false);
  const run = async() => {
    await new Promise(resolve => setTimeout(resolve, 1000 * 10));
    setSomething(true);
  }
  run();
}, []);
```

Is there anything that looks suspicious?

So what's going to happen when the component will be unmounted before our promise will be resolved? Do you think it will automatically stop execution? Or won't call setSomething? No and no, it will continue to execute the rest of this function's body like nothing happened and when setSomething(true) will be called - exception will be thrown by React.

You say, come on, it's not a big deal! Who care about it? Let's look at another example:

```javascript
useEffect(() => {
  const [ something, setSomething ] = useState(false);
  const run = async() => {
    const resp1 = await new Promise(resolve => /* CALL BACKEND */);
    // do something with the response
    const resp2 = await new Promise(resolve => /* CALL BACKEND */);
    // do something with the response
    // ... now call backend 3 - 4 times more

    // finally call setSomething, but it's already fucked
    // so it's doesn't really matter for our showcase
    setSomething(true);
  }
  run();
}, []);
```

So what happens now? We call backend over and over again without controlling if our component is already dismounted and finally as the cherry on the top of the cake we call setSomething that will throw exception. It's sooooo bad. So how to fix all this and make everyone happy?

## Ideal solution Promise.cancel

![](https://telegra.ph/file/dcf7e64cbbf45ce205c99.png)

If our component is dismounted then probably we don't need those promises anymore, also we could somehow cancel ongoing backend request and that would be ideal. Cancelling network request is actually possible (in case if you use fetch): [https://developer.mozilla.org/en-US/docs/Web/API/AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)

But we are working through chipmunk that doesn't have this functionality yet + you'll have to somehow collect all promises and somehow cancel them all and bla bla bla boring and complicated and won't work for us anyway, so let's think of something else.

## Just control everything

![](https://telegra.ph/file/07e60d28ebaffc7f1d00d.png)

Of course we could make something like this:

```javascript
useEffect(() => {
  let iAmStillWithYou = true;
  const run = async() => {

    const resp1 = await new Promise(resolve => /* CALL BACKEND */);
    if (!iAmStillWithYou) return;

    const resp2 = await new Promise(resolve => /* CALL BACKEND */);
    if (!iAmStillWithYou) return;

    const resp2 = await new Promise(resolve => /* CALL BACKEND */);
    if (!iAmStillWithYou) return;

    // and so on

  }
  run();
  return () => {
    iAmStillWithYou = false;
  }
}, []);
```

This will work, but requires to add check after each async call, which is again boring in my opinion but serves it's purpose well enough.

## What else we can do?

![](https://telegra.ph/file/eb9750a961fecc46945af.png)

Now it's time to make some dirty hacks! I know you've been waiting for it. So let's think, we are waiting for the promise to be resolved and the only things that can came out of it as a result is:

1. result itself (when promise is fullfiled)
2. nothing when you don't call resolve or reject your code will await forever
3. reject
So #1 is covered on the previous part of this article (when you return result from promise, then control if component is already dead and bla bla bla boring you remember). So what about the rest?

### Promises that just doesn't return

![](https://telegra.ph/file/18f86ab788e4bec76de24.png)

So what I'm taling about is this:

```javascript
const myPromise = new Promise(resolve => 3 + 2);
await myPromise;
// you'll never reach this point
```

So what if we could somehow simulate this sitation and make our backend calls promises to not return anything in case if the component is already dismounted? First of all we need some solution to control this, let's create wrapper that we will use to call our promises:

```javascript
const angryPromiseFactory = () => {

  let isCancelled = false;

  const wrapper = (value: any) => new Promise<any>(async (resolve) => {
    const result = await value;
    if (!isCancelled) resolve(result);
  });
  
  wrapper.cancel = () => {
    isCancelled = true;
  }

  return wrapper;
}
```


And this is how we use it:

```javascript
const [ something, setSomething ] = useState(false);
const asyncCall = React.useMemo(() => angryPromiseFactory(), []);

useEffect(() => {
  
  const run = async () => {
    const resp1 = await asyncCall(chipmunk.action(...));
    const resp2 = await asyncCall(chipmunk.action(...));
    // this will never be called if component is unmounted
    setSomething(true);
  };
  
  run();

  // once called
  // this will prevent all wrapped promises to return anything
  return asyncCall.cancel;

}, []);
```

So whenever our component will be dismounted no promise wrapped into asynCall will be resolved. Problem is solved, and we've got clean code, easy to understand, easy to write, easy to maintain.

### Is there anything that looks suspicious?
Wait a minute. So what's going to happen with our code if some of the promises there just doesn't return anything. Is this somehow magically will be interrupted or something like this? Unfortunatelly not. By creating the promises that doesn't return we just create bunch of memory leaks. We have a scope where we're waiting for the promise to be resolved. And this scope has it's own variables, definitions, closures and so on. This scope can't be destroyed because JS engine doesn't really know if our promise will ever be resolved. So this bunch of code will just sit there forever, because garbage collector won't be able to clean it because there are references to it and so on so on. What is most important here, is that our application will be more and more slow on each and every iteration and will consume more and more memory. Just google it or read stackoverflow: [https://stackoverflow.com/questions/20068467/does-never-resolved-promise-cause-memory-leak](https://stackoverflow.com/questions/20068467/does-never-resolved-promise-cause-memory-leak)

## Promises that rejects
So what if instead of returning nothing we will use another promise's feature called "reject"? Let's adjust our wrapper a little bit so instead of doing nothing on .cancel() it will reject everything:

```javascript
const angryPromiseFactory = () => {

  let isCancelled = false;
  let doReject: (reason: any) => any | null = null;

  const wrapper = (value: any) => new Promise<any>(async (resolve, reject) => {

    doReject = reject;
    const result = await value;
    if (!isCancelled) {
      doReject = null;
      resolve(result);
    }

  });
  
  wrapper.cancel = () => {
    isCancelled = true;
    if (doReject) {
      console.info('call doReject');
      doReject('PROMISE_RETURNED_AFTER_UNMOUNT');
      doReject = null;
    }
  }
  
  return asyncCallWrapper;
}
```

And this is how we use it:

```javascript
const [ something, setSomething ] = useState(false);
const asyncCall = React.useMemo(() => angryPromiseFactory(), []);

useEffect(() => {
  
  const run = async () => {
    const resp1 = await asyncCall(chipmunk.action(...));
    const resp2 = await asyncCall(chipmunk.action(...));
    // this will never be called if component is unmounted
    setSomething(true);
  };
  
  run();

  // once called
  // this will prevent all wrapped promises to return anything
  return asyncCall.cancel;

}, []);
```

So whenever our component will be dismounted it will call asyncCall.cancel that will call reject on currently executing promise. And that's it! No memory leak, because Promise is fullfiled, no complex conditions and so on.

The only thing is that now everytime when promise is rejected - annoying error message appears on the console, something like "Uncaught (in promise) ... ". Let's suppress it:

```javascript
window.addEventListener('unhandledrejection', event => {
  if (event.reason === 'PROMISE_RETURNED_AFTER_UNMOUNT') {
    console.log(`Suppressed the rejection '${event.reason}'`);
    event.preventDefault();
  }
});
```

And here we go, nice and clean no more annoying error messages in the console :D


---


**So what do you think about this solution? Maybe I'm making this too complicated and trying to solve problem that doesn't exist? How do you deal with such problem? Please ask questions or share your ideas in slack's discussion.**


**And thanks for reading!**