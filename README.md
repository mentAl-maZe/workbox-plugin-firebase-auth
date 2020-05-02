# Workbox plugin for firebase auth

This is a simple plugin for workbox strategies which adds an `Authorization: Bearer` header with the return value from [`firebase.User.getIdToken(true)`](https://firebase.google.com/docs/reference/js/firebase.User#getidtoken) to the [Request](https://developer.mozilla.org/en-US/docs/Web/API/Request) if a firebase User is authenticated (i.e. [`firebase.auth.Auth.onAuthStateChanged()`](https://firebase.google.com/docs/reference/js/firebase.auth.Auth#onauthstatechanged) returns a [`firebase.User`](https://firebase.google.com/docs/reference/js/firebase.User)).

This way you are able to cache authorized routes without relying on a session cookie or an API token.

## Usage

### Module

Use the module if:

- you are building your service worker [using a bundler](https://developers.google.com/web/tools/workbox/guides/using-bundlers).
- you are generating your service worker with [workbox-build](https://developers.google.com/web/tools/workbox/modules/workbox-build)

1. Add the dependency:

   ```sh
   npm i workbox-firebase-auth // or yarn add workbox-firebase-auth
   ```

2. Import the plugin and use it for your strategies

   Example:

   ```js
   import {registerRoute} from 'workbox-routing/registerRoute.mjs';
   import {NetworkFirst} from 'workbox-strategies/NetworkFirst.mjs';
   import FirebaseAuthPlugin from 'workbox-plugin-firebase-auth';

   registerRoute(
     /\.(?:png|gif|jpg|jpeg|svg)$/,
     new NetworkFirst({
       cacheName: 'authorizedApi',
       plugins: [
         new FirebaseAuthPlugin({
           firebase: {
             config: { /* your firebase config */ }
           }
         }),
       ],
     }),
   );
   ```

### CDN

If you are using [workbox-sw](https://developers.google.com/web/tools/workbox/modules/workbox-sw) to import workbox, you can use the [unpkg CDN](https://unpkg.com/) to import the plugin.  
It will then be available under the global variable `WorkboxPluginFirebaseAuth`.

Example:

```js
importScripts(
  'https://storage.googleapis.com/workbox-cdn/releases/5.1.2/workbox-sw.js',
  'https://unpkg.com/workbox-plugin-firebase-auth'
)

workbox.routing.registerRoute(
  /\.(?:png|gif|jpg|jpeg|svg)$/,
  new workbox.strategies.CacheFirst({
    cacheName: 'authorizedApi',
    plugins: [
      new WorkboxPluginFirebaseAuth({
        firebase: {
          config: { /* your firebase config */ }
        }
      }),
    ],
  }),
)
```

## Options

If your service worker is served from firebase hosting, associated with the firebase app you use to authorize users, you can omit configuration altogether.  
Otherwise the [`firebase.config`](#firebaseconfig) parameter is **REQUIRED**.

### firebase

This key is used to configure the firebase instance.

#### firebase.config

**Required:** If your service worker is NOT served from firebase hosting or if you use a different app to authorize users.  
**Type:** `object`

The [firebase config object](https://firebase.google.com/docs/web/setup?authuser=0#config-object) from the app that you use to authorize your users.

#### firebase.version

**Type:** `string`
**Default:** `7.14.2`

This key can be used to specify the firebase version to use.

> **Note:** This is currently a hard coded fallback and will manually be updated.  
> In the future this should use the version form the dependency used for development.  
> Sadly I haven't figured out how to achieve this yet (PRs welcome :sweat_smile:)

### awaitResponse

**Type:** `boolean`  
**Default:** `false`

If true the plugin will await the fetch to go through and check if the response has a 401 status before attaching the authorization and resending the request.  
Be aware that this happens before the response is passed to the caching strategy, please plan accordingly (e.g. a cache first strategy might serve authorized content to non authorized users).

> **Note:** Please make sure your server responds to unauthorized requests with a 401 status code, so that the plugin can correctly identify authorization failures.

### constraints

This key can be used to specify additional constraints on top of the route matcher.

#### constraints.types

**Type:** `string | string[]`
**Default:** `['*']`

This can be used to authorize only requests that accept certain types of responses (e.g. `application/json`)

> **Note:** This simply matches the entries from the `Accept` request header against the passed array/string.  
> As of yet group matching is not supported (e.g. `text/*` will not match `text/html`)

#### constraints.https

**Type:** `boolean`
**Default:** `false`

Only allow requests from secure origins (`https://` or `localhost`) to be authorized.

#### constraints.ignorePaths

**Type:** `(string | RegExp)[]`
**Default:** `[]`

Paths to ignore when authorizing requests.  

> **Note:** Checks against the pathname of the request (e.g. `/api/some-resource`)  
> If the argument is a `string` a request will be ignored if the pathname starts with that `string`.  
> If the argument is a `RegExp` the leading `'/'` will be stripped from the pathname and it will be checked against the `RegExp`.
