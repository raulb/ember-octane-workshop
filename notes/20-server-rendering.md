# Server Rendering with Fastboot

Ember's server rendering technology is called "Fastboot". It runs on Node.js. Effectively, it's a small distributed system consisting of:

- 1 main server process that receives requests
- N workers, each of which maintains a "warm" ember app, ready to emit HTML for a given URL.

The idea is that we should _not_ be writing two seperate apps — a single Ember app should work in the browser *and* on the server. In order to do this, we'll need to stick to the _overlap_ of browser and Node.js APIs.

![js environments](./img/20-server-rendering/js-envs.png)

First, let's install fastboot to enable server-rendering

```
ember install ember-cli-fastboot
```

Open your [`config/environment.js`](../config/environment.js), and add a `fastboot` object as a top-level property of the `ENV` property you'll find there.

```diff
+    fastboot: {
+      hostWhitelist: [/localhost/],
+    },
     APP: {
```

We'll have to refactor our `auth` service so that it doesn't use any browser-specific APIs.

To do this we can install `ember-cookies`, a unified abstraction for dealing with cookies both in node and in a browser environment.

```
ember install ember-cookies
```

Now let's stop and restart ember-cli -- we should see that the app is now being served with fastboot.

Next, we'll have to make some adjustments to [`app/services/auth.js`](../app/services/auth.js). Begin by injecting the `cookies` service:

```ts
@service cookies;
```

Update the `loginWithUserId` function so it writes to a cookie instead of `localStorage`

```diff
   async loginWithUserId(userId) {
-    window.localStorage.setItem(AUTH_KEY, userId);
+    this.cookies.write(AUTH_KEY, userId, {path: '/'});
     this.router.transitionTo('teams');
   }
```

Update the `currentUserId` getter so it reads from a cookie

```diff
   get currentUserId() {
-    return window.localStorage.getItem(AUTH_KEY);
+    return this.cookies.read(AUTH_KEY);
   }
```

And update `logout` so it clears the value from the cookie instead of`localStorage`:

```diff
   logout() {
-    window.localStorage.removeItem(AUTH_KEY);
+    this.cookies.write(AUTH_KEY, null, {path: '/'});
     this.router.transitionTo('login');
   }
 }
```

Let’s take the app for a spin and see if we can still log in and out. If we’re very brave we can also try disabling JavaScript in the browser!

## 🙌

Nice! This exercise was a quick introduction to Fastboot and some of the extra stuff we have to consider when writing apps to work in the browser and on the server.
