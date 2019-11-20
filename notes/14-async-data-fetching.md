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
3. We fetch the channel data from the API using the two IDs above.
4. We convert the JSON body of the response into data.
5. We return the data for use in the template.

Now that we’ve added all real data-fetching logic to our routes, we’ll turn out attention to the auth service.
Let’s open [`app/services/auth.js`](../app/services/auth.js) and make the following modifications.

Add the required imports for using [`tracked`](https://glimmerjs.com/guides/tracked-properties) and `fetch`:

```js
import { tracked } from '@glimmer/tracking';
import fetch from 'fetch';
```

Then define a `tracked` property, `currentUser`:

```js
@tracked currentUser = null;
```

Now add the `loadCurrentUser` method that we referred to in the `teams` route.

```js
async loadCurrentUser() {
  if (!this.currentUserId) {
    return;
  }

  const response = await fetch(`/api/users/${this.currentUserId}`);
  const user = await response.json();

  this.currentUser = user;
}
```

## Display fetched data

We are now done with all the data fetching needed for this exercise. Now it’s time to display them in our templates.

In [`app/templates/login.hbs`](../app/templates/login.hbs), pass the model data into the `<LoginForm />` component:

```hbs
<LoginForm @users={{this.model}} />
```

In [`app/components/login-form.hbs`](`../app/components/login-form.hbs`), replace the latter two hard-coded `<option>` elements with dynamic ones:

```hbs
{{#each @users as |user|}}
  <option
    selected={{eq this.userId (concat '' user.id)}}
    value={{user.id}}
  >
    {{user.name}}
  </option>
{{/each}}
```

Let’s unpack what’s going on here:

1. We render an `<option>` for each `user` in the `@users` array.
2. If the `id` of a user matches `this.userId` then we mark it as selected.
3. We use a little trick (`(concat '' user.id)`) to convert `user.id` to a string.
4. We do this to ensure the comparision to `this.userId` works correctly.
5. `(concat '' user.id)` is equivalent to `'' + user.id` in JavaScript.

## Tests

We’ve added a new method to the auth service, so we need to add a corresponding method to StubbedAuthService.

In [`tests/test-helpers/auth-service.js`](../tests/test-helpers/auth-service.js) add a stub definition for `loadCurrentUser`:

```js
loadCurrentUser() {
  if (!this.isAuthenticated) {
    return;
  }

  this.currentUser = {
    id: this.currentUserId,
    name: 'Mike North',
  };
}
```

We also need to update the tests for `<LoginForm>`.

Update [`tests/integration/components/login-form-test.js`](../tests/integration/components/login-form-test.js), replacing all instances of this:

```hbs
await render(hbs`<LoginForm />`);
```

With this:

```js
this.set('myUsers', [
  { id: 1, name: 'Sample McFixture' },
  { id: 2, name: 'Testy Assertington' },
]);

await render(hbs`<LoginForm @users={{this.myUsers}}/>`);
```

Finally, update the assertions to expect the new data. That is, replace `'Testy Testerson'` and `'Sample McData'` with `'Sample McFixture'` and `'Testy Assertington'`.

## 🙌

Awesome — we’ve introduced real data fetching to our app!