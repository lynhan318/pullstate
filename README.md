### pullstate

> Ridiculously simple state stores with performant retrieval anywhere
> in your React tree using the wonderful concept of React hooks!

* ~1.87KB minified and gzipped! (excluding Immer and React)
* Built with Typescript, providing a great dev experience if you're using it too
* Provides `<InjectStoreState>` component for those who don't like change 🐌
* Uses [immer](https://github.com/mweststrate/immer) for state updates - easily and safely mutate your state directly!
* **NEW** - [Create async actions](#async-actions) and use hooks to watch their state. Pullstate's version of React suspense!

_Originally inspired by the now seemingly abandoned library - [bey](https://github.com/jamiebuilds/bey). Although substantially
different now- with Server-side rendering and Async Actions built in! Bey was in turn inspired by
[react-copy-write](https://github.com/aweary/react-copy-write)._

Try out a quick example:

[![Edit Pullstate Client-only Example](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/myvj8zzypp)

## **Let's dive right in**

```bash
yarn add pullstate
```

After installing, lets define a store by passing an initial state to `new Store()`:

```javascript
import { Store } from "pullstate";

export const UIStore = new Store({
  theme: {
    mode: EThemeMode.DARK,
  },
  message: `What a lovely day`,
});
```

Then, in React, we can start using the state of that store using a simple hook `useStoreState()`:

```jsx
import { UIStore } from "./stores/UIStore";
import { useStoreState } from "pullstate";

const App = () => {
  const theme = useStoreState(UIStore, s => s.theme);

  return (
    <div className={`app ${theme}`}>
      <button
        onClick={() => {
          UIStore.update(s => {
            s.theme.mode = theme.mode === EThemeMode.DARK ? EThemeMode.LIGHT : EThemeMode.DARK;
          });
        }}
      >
        Switch it up!
      </button>
    </div>
  );
};
```

Notice, that we also made use of `update()`, which allows us to update our stores' state anywhere we please - (literally anywhere in your JS code, not only inside React components - __but if you are using Server Rendering, see below 👇__) - over here we simply did it inside a click event to change the theme.

Also notice, the second argument to `useStoreState()`:

```javascript
const theme = useStoreState(UIStore, s => s.theme);
```

This selects a sub-state within our store. This ensures that this specific "hook" into our store will only update when that specific return value is actually changed in our store. This enhances our app's performance by ignoring any changes in the store this component does not care about, preventing unnecessary renders.

**E.g** If we had to update the value of `message` in the `UIStore`, nothing would happen here since we are only listening
for changes on `theme`.

If you want you can leave out the second argument altogether:

```jsx
const storeState = useStoreState(UIStore);
```

This will return the entire store's state - and listen to all changes on the store - so it is generally not recommended.

To listen to more parts of the state within a store simply pick out more values:

```javascript
const { theme, message } = useStoreState(UIStore, s => ({ theme: s.theme, message: s.message }));
```

Lastly, lets look at how we update our stores:

```javascript
UIStore.update(s => {
  s.theme.mode = theme.mode === EThemeMode.DARK ? EThemeMode.LIGHT : EThemeMode.DARK;
});
```

Using the power of [immer](https://github.com/mweststrate/immer), we update a store by calling a function called `update()` on it. The argument is the updater function, which is given the current state of our store to mutate however we like! For more information on how this works, go check out [immer](https://github.com/mweststrate/immer). Its great.

And that's pretty much it!

As an added convenience (and for those who still enjoy using components directly for accessing these things), you can also work with state using the `<InjectStoreState>` component, like so:

```jsx
import { InjectStoreState } from "pullstate";

// ... somewhere in your JSX :
<InjectStoreState store={UIStore} on={s => s.message}>{message => <h2>{message}</h2>}</InjectStoreState>
```

## Server Rendering

The above will sort you out nicely if you are simply running a client-rendered app. But Server Rendering is a little more involved (although not much).

### Create a central place for all your stores using `createPullstate()`

```javascript
import { UIStore } from "./stores/UIStore";
import { UserStore } from "./stores/UserStore";
import { createPullstate } from "pullstate";

export const PullstateCore = createPullstate({
  UIStore,
  UserStore,
});
```

You pass in the stores you created before. This creates a centralized object from which Pullstate can instantiate your state before each render.

### Using your stores on the server

```jsx
import { PullstateCore } from "./state/PullstateCore";
import ReactDOMServer from "react-dom/server";
import { PullstateProvider } from "pullstate";

// A server request
async function someRequest(req) {
  const instance = PullstateCore.instantiate({ ssr: true });

  const user = await UserApi.getUser(id);

  instance.stores.UserStore.update(userStore => {
    userStore.userName = user.name;
  });

  const reactHtml = ReactDOMServer.renderToString(
    <PullstateProvider instance={instance}>
      <App />
    </PullstateProvider>
  );

  const body = `
<script>window.__PULLSTATE__ = '${JSON.stringify(instance.getPullstateSnapshot())}'</script>
${reactHtml}`;

  // do something with the generated html and send response
}
```

* Instantiate fresh stores before each render using `instantiate()` - passing in `ssr: true`
* Manipulate your state directly during your server's request by using the `stores` property of the instantiated object.
* Notice we called `update()` directly on the `UserStore` here - this is a convenience method (which is actually available on all stores).
* We pass our pullstate instance into the rendering function. We use `<PullstateProvider>` to do so, providing the `instance`.
* Lastly, we need to return this state to the client somehow. Here we call `getPullstateSnapshot()` on the instance and set it on `window.__PULLSTATE__`, to be parsed and hydrated on the client.

### Client state hydration

```jsx
const hydrateSnapshot = JSON.parse(window.__PULLSTATE__);

const instance = PullstateCore.instantiate({ ssr: false, hydrateSnapshot });

ReactDOM.render(
  <PullstateProvider instance={instance}>
    <App />
  </PullstateProvider>,
  document.getElementById("react-mount")
);
```

* We create a new instance on the client using the same method as on the server, except this time we can pass the `hydrateSnapshot` and `ssr: false`, which will instantiate our new stores with the state where our server left off.

### Using our stores throughout our React app

So now that we have our stores properly injected into our react app through `<PullstateProvider>`, we need to actually make use of them correctly. Because we are server rendering, we can't use the singleton-type stores we made before - we need to target these injected store instances directly.

For that we need a new hook - `useStores()`.

This hook uses React's context to obtain the current render's stores, given to us by `<PullstateProvider>`.

Lets refactor the previous client-side-only example to work with Server Rendering:

```jsx
import { useStoreState, useStores } from "pullstate";

const App = () => {
  const { UIStore, UserStore } = useStores();
  const theme = useStoreState(UIStore, s => s.theme);
  const userName = useStoreState(UserStore, s => s.userName)

  return (
    <div className={`app ${theme}`}>
      <button
        onClick={() => {
          UIStore.update(s => {
            s.theme.mode = theme.mode === EThemeMode.DARK ? EThemeMode.LIGHT : EThemeMode.DARK;
          });
        }}
      >
        Switch it up, {userName}!
      </button>
    </div>
  );
};
```

Basically, all you need to do is replace the import

```

import { UIStore } from "./stores/UIStore";
```

with the context hook:

```

const { UIStore } = useStores();
```

As a **TypeScript** convenience, there is a method on your created `PullstateCore` object of all your stores also named `useStores()` which will give you all the typing goodness since it knows about the structure of your stores:

```

const { UIStore, UserStore } = PullstateCore.useStores();
```

### Last note about Server Rendering

On the client side, when instantiating your stores, you are now instantiating with your "origin" stores by passing the `ssr: false` like so:

```jsx
const hydrateSnapshot = JSON.parse(window.__PULLSTATE__);

const instance = PullstateCore.instantiate({ ssr: false, hydrateSnapshot });

ReactDOM.render(
  <PullstateProvider instance={instance}>
    <App />
  </PullstateProvider>,
  document.getElementById("react-mount")
);
```

Basically, what this does is re-uses the exact stores that you originally created.

This allows us to directly update those original stores on the client and we will receive updates as usual. On the server, calling `instantiate({ ssr: true })` creates a fresh copy of your stores (which is required because each client request needs to maintain its own state), but on the client code - its perfectly fine to directly update your created stores because the state is contained to that client alone.

For example, you could now do something like this:

```jsx
import { UIStore, GraphStore } from "./stores"

async function refreshGraphData() {
  UIStore.update(s => { s.refreshing = true; });
  const newData = await GraphDataApi.getNewData();
  GraphStore.update(s => { s.data = newData; });
  UIStore.update(s => { s.refreshing = false; });
}
```

⚠ __Caution Though__ - While this allows for much more ease of use for playing around with state on the client, you must make sure that these state updates are _strictly_ client-side only updates - as they will not apply on the server and you will get unexpected results. Think of these updates as updates that will run after the page has already loaded completely for the user (UI responses, dynamic data loading, loading screen popups etc.).

## Async Actions

More often than not, our stores do not exist in purely synchronous states. We often need to perform actions asynchronously, such as pulling data from an API.

* It would be nice to have an easy way to keep our view up to date with the state of these actions **without putting too much onus on our stores directly** which quickly floods them with variables such as `userLoading`, `updatingUserInfo`, `userLoadError` etc - which we then have to make sure we're handling for each unique situation - it just gets messy quickly.

* Having our views naturally listen for and initiate asynchronous state gets rid of a lot of boilerplate which we would usually need to write in `componentDidMount()` or the `useEffect()` hook.

* There are also times where we are **server-rendering** and we would like to resolve our app's asynchronous state before rendering to the user. And again, without having to run something manually (and deal with all the edge cases manually too) like we saw in the server rendering section above:

```jsx
const user = await UserApi.getUser(id);

instance.stores.UserStore.update(userStore => {
userStore.userName = user.name;
});
```

:point_up: And that's without dealing with errors and the like...

Pullstate provides a way to do this through **Async Actions**.

### Create an Async Action

If you are creating a client-only app, you can create an async action like so:

```jsx
import { createAsyncAction } from "pullstate";
import { UserStore } from "./stores/UserStore";

const GetUserAction = createAsyncAction(async ({ userId }) => {
  const user = await UserApi.getNewUser(userId);
  UserStore.update(s => {
    s.user = user;
  });
  return true;
});
```

If you are using server rendering, create them like this (using your `PullstateCore` object with all your stores):

```jsx
const GetUserAction = PullstateCore.createAsyncAction(async ({ userId }, { UserStore }) => {
  const user = await UserApi.getNewUser(userId);
  UserStore.update(s => {
    s.user = user;
  });
  return true;
});
```

Let's look closer at the actual async function which is passed in to `createAsyncAction()`:

* **The first argument** to this function is important in that it not only represents the variables passed into your action for each asynchronous scenario, but these arguments create a **unique "fingerprint"** each time the action is called which helps Pullstate keep track of actions being called.

* In that sense, these actions should be "pure" per the arguments you pass in - in this case, we pass an object containing `userId` and we expect to return exactly that user from the API.

* :warning: Pulling `userId` from somewhere else, such as directly within your store, will cause different actions for different user ids to be seen as the same! (because their "fingerprints" are the same) This will cause caching issues - so **always have your actions defined with as many arguments which identify that single action as possible**!

* The function should return a **non-null** value - as a response of `null` is the default when the function throws an error. If your actions have potential for error, is recommended to make use of `try {} catch {}` blocks and returning custom error states in your return value to be dealt with gracefully (e.g. `return { isError: true, tag: "USER_NOT_FOUND" }`).

* **The Pullstate Way** :tm:, is keeping your state in your stores as much as possible - hence we don't actually return the new user object, but update our `UserStore` along the way in the action (this also means that during a single asynchronous action, we can actually have our app update and react multiple times). In this case we just return `true`.

Notice that the async function looks slightly different when using server-side rendering:

* Your Pullstate stores are passed into the function as the second argument - you must use these stores directly because any asynchrounous state that we need to resolve on the server before rendering needs to use the stores provided by `<PullstateProvider>`, which will be these.

### Using Async Actions

There are four ways to use Async Actions in your code:

#### Watch the state of an Async Action

```jsx
const [started, finished, result, updating] = GetUserAction.useWatch({ userId });
```

* For the sake of being
