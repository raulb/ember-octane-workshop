# Async Data Fetching

In this exercise we will be replacing the hard coded data that we added in previous exercises with data fetched from server.

For fetching data from server, we are going to use the [fetch](https://developers.google.com/web/updates/2015/03/introduction-to-fetch) method that is natively available in most modern browsers. In our app, we use the [ember-fetch](https://github.com/ember-cli/ember-fetch) ember addon that uses a polyfill for fetch method, in browsers that don't support the fetch method (and even in Node). You can find more info about browser support for fetch method [here](https://caniuse.com/#feat=fetch).

Let's get started!

## Fetch data from server

First, in [`app/routes/login.js`](../app/routes/login.js) let’s fetch the list of users. Start by adding an import for the `fetch` method alongside the other imports:

```js
import fetch from 'fetch';
```

Then add a `model` hook that returns the list of users fetched from server:

```js
async model() {
  const response = await fetch('/api/users');
  const users = await response.json();

  return data;
}
```

In [`app/routes/teams.js`](../app/routes/teams.js), let’s fetch data about the logged-in user.

Once again, import `fetch` at the top of the file:

```js
import fetch from 'fetch';
```

Then replace the existing `beforeModel` hook with this:

```js
async beforeModel(transition) {
  if (this.auth.isAuthenticated) {
    await this.auth.loadCurrentUser();
  } else {
    this.transitionTo('login');
  }
}
```

In this hook, we:

1. Check if the current user is logged-in.
2. If they are, then we load their data
3. If not, we transition to the login route

Now let’s load the list of teams. Replace the existing `model` hook with this:

```js
async model() {
  const response = await fetch('/api/teams');
  const teams = await resp.json();

  return teams;
}
```

Now that we are fetching data from the server, we don't need the `ALL_TEAMS` array anymore so let’s delete it.

Next let’s add real data-fetching to [`app/routes/teams/team.js`](../app/routes/teams/team.js).

Replace the import for the hard coded array, with import for `fetch`.

Delete the `ALL_TEAMS` import:

```js
import { ALL_TEAMS } from '../teams';
```

And import `fetch` in its place:

```js
import fetch from 'fetch';
```

Then, replace the `model` with this:

```js
async model({ teamId }) {
  const response = await fetch(`/api/teams/${teamId}`);
  const team = await response.json();

  return team;
}
```

Next, let’s add data-fetching to [`app/routes/teams/team/channel.js`](../app/routes/teams/team/channel.js).

Import `fetch`:

```js
import fetch from 'fetch';
```

Then replace the `model` hook with this:

```js
async model({ channelId }) {
  const { teamId } = this.paramsFor('teams.team');
  const response = await fetch(`/api/teams/${teamId}/channels/${channelId}`);
  const channel = await response.json();

  return channel;
}
```

There’s a bit more going on here, so let’s step through it:

1. We receive `channelId` as a param because it’s a dynamic segment in our path (`:channelId`).
2. We grab the `teamId` param from our parent route by calling `this.paramsFor('teams.team')`.
3. We fetch the channel data from the API using the two IDs above
4. We convert the JSON body of the response into data
5. We return the data for use in the template.

Next, modify the `auth` service defined at [`../app/services/auth.js`](../app/services/auth.js).

Add the required imports for using [`tracked`](https://glimmerjs.com/guides/tracked-properties) properties and `fetch` method:

```diff
+   import { tracked } from '@glimmer/tracking';
+   import fetch from 'fetch';
```

Then define a `tracked` property, `currentUser`:

```diff
+   @tracked currentUser = null;
```

Modify the `loginWithUserId` method as an `async` method, since it is going to be called with `await` keyword.

```diff
-   loginWithUserId(userId) {
+   async loginWithUserId(userId) {
```

Add the definition for `loadCurrentUser` method that we used earlier in the exercise, in the `teams` route, defined at [`../app/routes/teams.js`](../app/routes/teams.js).

```diff
+   async loadCurrentUser() {
+     const { currentUserId } = this;
+     if (!currentUserId) return;
+     this.currentUser = await (await fetch(
+       `/api/users/${currentUserId}`
+     )).json();
+   }
```

## Display fetched data

We are now done with all the data fetching needed for this exercise. Now its time to display them in our templates.

In the [`login`](../app/templates/login.hbs) template, pass the model data into the `login-form` component. Note here that the login-form component used here, is following angle bracket syntax.

```diff
-   <LoginForm />
+   <LoginForm @users={{this.model}}/>
```

In the [`login-form`](`../app/components/login-form.hbs`) component template, replace the hard coded values in the user selection dropdown to dynamic values fetched from server. Here `@users` contains the dynamic data that was feed into it from the `login.hbs` template.

```diff
-   <option selected={{eq this.userId "1"}} value="1">Testy Testerson</option>
-   <option selected={{eq this.userId "2"}} value="2">Sample McData</option>
+   {{#each @users as |user|}}
+     <option selected={{eq this.userId (concat '' user.id)}} value={{user.id}}>{{user.name}}</option>
+   {{/each}}
```

## Test helper

In the `auth` service defined at [`../app/services/auth.js`](../app/services/auth.js), we added some new methods. Lets update the test helper present at [`../tests/test-helpers/auth-service.js`](../tests/test-helpers/auth-service.js) accordingly. This test helper makes sure every time tests are run, there is no state leakage across tests, and application state stored on the auth service is reset, before each test is run.

Add the import for `action` at the top of the file, in the imports section.

```diff
+   import { action } from '@ember/object';
```

Add definition for `loginWithUserId` method:

```diff
+   async loginWithUserId(id) {
+     this.currentUserId = id;
+     await this.loadCurrentUser();
+     this.router.transitionTo('teams');
+   }
```

Note that in the above updated method definition for `loginWithUserId`, we added a call to the new method, `loadCurrentUser()`. So lets add the definition for `loadCurrentUser` method accordingly.

```diff
+   async loadCurrentUser() {
+     if (!this.isAuthenticated) return;
+     this.currentUser = {
+       id: this.currentUserId,
+       name: 'Mike North',
+     };
+   }
```

```diff
+   @action
+   logout() {
+     this.currentUserId = null;
+     this.currentUser = null;
+     this.router.transitionTo('login');
+   }
```

## Tests

Update the tests for the [`login-form`](../tests/integration/components/login-form-test.js) component to reflect the changes we made earlier in this exercise, to replace all occurrences of hard coded data with data fetched from server.

```diff
-   await render(hbs`<LoginForm />`);
+   this.set('myUsers', [
+     { id: 1, name: 'Sample McFixture' },
+     { id: 2, name: 'Testy Assertington' },
+   ]);
+
+   await render(hbs`<LoginForm @users={{this.myUsers}}/>`);
```

```diff
-   ['Login', 'Select a user', 'Testy Testerson', 'Sample McData']
+   ['Login', 'Select a user', 'Sample McFixture', 'Testy Assertington']
```

```diff
+   this.set('myUsers', [
+     { id: 1, name: 'Sample McFixture' },
+     { id: 2, name: 'Testy Assertington' },
+   ]);
```

```diff
-   await render(hbs`<LoginForm />`);
+   await render(hbs`<LoginForm @users={{this.myUsers}}/>`);
```

```diff
-   'Testy Testerson',
-   'Sample McData',
+   'Sample McFixture',
+   'Testy Assertington',
```

And that's it. You are now done with this exercise!
