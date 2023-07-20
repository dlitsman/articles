# Using Screen Wake Lock API in React. Modern Browser API to prevent device screen lock

Devices usually turn off displays after some time to improve power usage, improve security, and prolong life of hardware in general. However, this behaviour is not always providing best UX for users. In some cases browsers are smart enough to prevent sleep, by using heuristics like playing video. However, in some cases it is a perfectly valid case to prevent device from locking (e.g., navigation like system, photo gallery, etc.)

Even though this API is not fully part of the standard, and in the [Working Draft](https://www.w3.org/TR/screen-wake-lock/) state it has good browser coverage. Especially since version 16.4 it is supported in Safari. So currently it is supported in most of major browsers except Firefox.

![image](./imgs/browser-support.png)

Let's take a closer look how to use it in practice and what are the limitations.

## Screen Wake Lock API

There are 3 important limitations when you are allowed to use this api
1. Your app should be in an active tab
2. You app should be served from [Secure Context](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts)
3. Using of this feature is not blocked by [Permissions Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Permissions_Policy)

API itself is quite straightforward and consists of two steps: requesting lock and releasing it. The API itsef is Promise based and you need to wait for promise resolution to acquire the lock.

There are 2 main parts in the API
- **WakeLock** - Interface that can be used to request the lock from the browser
- **WakeLockSentinel** - Interface to the underlying lock that can be used to release the lock

So let's connect all the pieces together 

### Requesting a lock

To request a lock we need to use `navigator.wakelock` and invoke `request()` method. You need to provide type of lock, but only `screen` type is supported at this moment.

```js
const requestScreenWakeLock = async () => {
  // Check if wakelock API is available
  if (!("wakeLock" in navigator)) {
    console.error("Screen Wake Lock API is not supported by the browser")  
  }

  try {
    const wakeLock = await navigator.wakeLock.request("screen");
  } catch (err) {
    // There are various reason why lock might not be acquired such as tab is not active, low battery on a device, permissions
    console.error(`WakeLock request error: ${err.message}`);
  }
};
```

### Releasing a lock

Once you have a `WakeLockSentinel` from `request()` API you can use it to release a lock. For this you can simple call `release()` method on it and wait for the promise to resolve. You can also check status of a lock in a `released` property and `type`. Another helpful method on a lock you can listen for `release` event to add custom logic in this case.

The simplest snippet

```js
try {
  await wakeLock.release();
} catch (err) {
  console.error(`WakeLock release error: ${err.message}`);
}
```

Even though API itself seems quite straightforward, there are some caveats to consider when implementing it yourself. For example, as lock will be auto-released when browser is not active anymore you will need to take care of it yourself. In the next section I'll show how to implement it in React and share the npm package that I created to simplify this process

## Using Screen Wake Lock API in a React

There are some existing libraries you can use that have some wrappers around Wake Lock. However, all of them have important downside, they provide imperative way of requesting Wake Lock, meaning you will have to manually request/release and manage lock lifecycle, as well as making sure to re-request it when browser lost focus. 

In this article, I'm proposing declarative way to enable this API. This will help hide all of the complexities and provide a single argument if lock should be active or not. So ideal API will look something like that

```js
const [shouldLock, setShouldLock] = useState(true);

// Simple boolean that decides if we need to acquire a lock. useWakeLock will deal with all of the complexities behind the scene. 
// In case of success it will return true or false
const isLocked = useWakeLock(shouldLock);

```

So without further ado let's take a look how we can implement it

```ts
export function useWakeLock(enabled: boolean) {
   // TODO Add snippet once done
}
```

This is simplified version of a snippet that you can copy-paste and adopt for your needs. However, if you want more controls out of the box you can use `react-use-wake-lock` npm library that has extra configuration and zero dependencies.

### How to use react-use-wake-lock

1. Install with `npm install react-use-wake-lock`
2. Import in your project

```js
import {useWakeLock} from `react-use-wake-lock`

function SomeComponent() {
    // ...
    const lockConfig = useMemo(({
        onError: (err: Error, flow: 'request' | 'release') => {console.error(`Error during ${flow}: ${err.message}`)},
        onLock: (lock: WakeLockSentinel) => {console.info('Lock acquired')},
        onRelease: (lock: WakeLockSentinel) => {console.info('Lock released')},
    }), []);
    const {isSupported, isLocked} = useWakeLock(shouldLock, lockConfig)
    // ...
}
```

# Conclusion

Using modern browser API helps us build much better UX and bring app like experience to the web. You can find the full source code as well as demo and installation instructions for this example on GitHub https://github.com/dlitsman/react-wake-lock.