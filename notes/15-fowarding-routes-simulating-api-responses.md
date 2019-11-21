# Redirecting Between Routes & Simulating API Responses

In this exercise, we will focus on:

- Conditionally redirecting to different routes
- Simulating API responses for tests
- Adding new tests

## Redirecting Between Routes

Right now we don’t see anything if we visit the home route (plain old `/`). Let’s make the app redirect us to a more useful route instead.

We’re going to put this logic in the `index` route. `index` routes are rendered by default when no explicit leaf route is specified, just like an `index.html` file on a traditional web server.


```
/                    (application)
│
├─ /                 (index)
│
└─ teams             (teams)
   │
   ├─ /              (team.index)
   │
   └─ :teamId        (teams.team)
      │
      ├─ /           (team.team.index)
      │
      └─ :channelId  (teams.team.channel)
 
```

Let’s generate our index route:

```sh
ember g route index
```

Now replace the contents of `app/routes/index.js` with the following:


```js
import Route from '@ember/routing/route';
import { inject as service } from '@ember/service';
import AuthService from 'shlack/services/auth';

export default class IndexRoute extends Route {

  @service auth;

  beforeModel() {
    if (this.auth.isAuthenticated) {
      this.transitionTo('teams');
    } else {
      this.transitionTo('login');
    }
  }

}
```

Similarly, when we visit just `/teams` or just `/teams/some-id` then the app should take us to the logical route.

For the `teams` route, if there is at least 1 team then let’s display the first one. Otherwise, let’s display a suitable empty state.

Let’s generate `teams/index`:

```sh
ember g route teams/index
```

Now replace the contents of `app/routes/teams/index.js` with the following:

```js
import Route from '@ember/routing/route';

export default class TeamsIndexRoute extends Route {

  beforeModel() {
    const [team] = this.modelFor('teams');

    if (team) {
      this.transitionTo('teams.team', team.id);
    }
  }

}
```

We’re going to use the same trick for `team/index`, but this time redirecting to the first channel in the team. First, generate the route:

```sh
ember g route teams/team/index
```

Then replace the contents

```js
import Route from '@ember/routing/route';

export default class TeamsTeamIndexRoute extends Route {

  beforeModel() {
    const { channels: [channel] } = this.modelFor('teams.team');

    if (channel) {
      this.transitionTo('teams.team.channel', channel.id);
    }
  }

}
```

## Displaying Empty States

Now that we have all these `index` templates, let’s display some empty states on them.

Replace the contents of `app/templates/teams/index.hbs` with the following:

```hbs
<div class="mx-auto">
  <div class="flex justify-center flex-row w-full leading-loose text-3xl">
    No teams in this app
  </div>
</div>
```

Let’s do the same for `app/templates/teams/team/index.hbs`:

```hbs
<div class="mx-auto">
  <div class="flex justify-center flex-row w-full leading-loose text-3xl">
    No channels in this team
  </div>
</div>
```

## Tests

We modified the team and teams routes, for conditionally forwarding a user to different routes. So, let's update the tests accordingly.

In the `login-test` test located at [`../tests/acceptance/login-test.js`](../tests/acceptance/login-test.js):

```diff
-   assert.equal(currentURL(), '/teams');
+   assert.ok(currentURL().startsWith('/teams'));
```

```diff
-   assert.equal(currentURL(), '/teams');
+   assert.ok(currentURL().startsWith('/teams'));
```

In the `logout-test` located at [`../tests/acceptance/logout-test.js`](../tests/acceptance/logout-test.js):

```diff
-   await visit('/teams/li'); // Go to a URL
+   await visit('/teams'); // Go to a URL
```

```diff
-   assert.equal(currentURL(), '/teams/li'); // Make sure we've arrived
+   assert.ok(currentURL().startsWith('/teams/li')); // Make sure we've arrived
```

We also added some routes in this exercise. So let's add some unit tests for the same.

For the `index` route, add an unit test at [`../tests/unit/routes/index-test.js`](../tests/unit/routes/index-test.js):

```diff
+   import { module, test } from 'qunit';
+   import { setupTest } from 'ember-qunit';
+
+   module('Unit | Route | index', function(hooks) {
+     setupTest(hooks);
+
+     test('it exists', function(assert) {
+       let route = this.owner.lookup('route:index');
+       assert.ok(route);
+     });
+   });
```

For the `teams` route, add an unit test at [`../tests/unit/routes/teams/index-test.js`](../tests/unit/routes/teams/index-test.js):

```diff
+   import { module, test } from 'qunit';
+   import { setupTest } from 'ember-qunit';
+
+   module('Unit | Route | teams/index', function(hooks) {
+     setupTest(hooks);
+
+     test('it exists', function(assert) {
+       let route = this.owner.lookup('route:teams/index');
+       assert.ok(route);
+     });
+   });
```

And finally, for the `channels` route, add an unit at [`../tests/unit/routes/teams/team/index-test.js`](../tests/unit/routes/teams/team/index-test.js):

```diff
+   import { module, test } from 'qunit';
+   import { setupTest } from 'ember-qunit';
+
+   module('Unit | Route | teams/team/index', function(hooks) {
+     setupTest(hooks);
+
+     test('it exists', function(assert) {
+       let route = this.owner.lookup('route:teams/team/index');
+       assert.ok(route);
+     });
+   });
```

## Simulating Responses In Tests

Now, let's add some tests for testing the route forwarding implementation, that we added earlier in this exercise.

In the acceptance tests, we will be simulating API responses using [pretender](https://github.com/pretenderjs/pretender)(a mock server library), which we include in this application, by using the [ember-cli-pretender](https://github.com/rwjblue/ember-cli-pretender) addon.

Create a new file for adding acceptance test at [`../tests/acceptance/forwarding-routes-test.js`](../tests/acceptance/forwarding-routes-test.js):

```diff
+   import { module, test } from 'qunit';
+   import { visit, currentURL } from '@ember/test-helpers';
+   import { setupApplicationTest } from 'ember-qunit';
+   import StubbedAuthService from '../test-helpers/auth-service';
+   import Pretender, { ResponseHandler } from 'pretender';
+
+   /**
+    *
+    * @param {any} body
+    * @returns {ResponseHandler}
+    */
+   function jsonResponse(body) {
+     return function() {
+       return [200, {}, JSON.stringify(body)];
+     };
+   }
+
+   /**
+    * @this {Pretender}
+    */
+   function setupServer() {
+     this.get(
+       '/api/users',
+       jsonResponse([
+         { id: 1, name: 'Sample McFixture' },
+         { id: 2, name: 'Testy Assertington' },
+       ])
+     );
+     this.get(
+       '/api/teams',
+       jsonResponse([
+         {
+           id: 'gh',
+           name: 'GitHub',
+         },
+       ])
+     );
+     this.get(
+       '/api/teams/gh',
+       jsonResponse({
+         id: 'gh',
+         name: 'GitHub',
+         channels: [
+           {
+             id: 'prs',
+             name: 'Pull Requests',
+           },
+         ],
+       })
+     );
+     this.get(
+       '/api/teams/gh/channels/prs',
+       jsonResponse({
+         id: 'prs',
+         teamId: 'gh',
+         name: 'Pull Requests',
+       })
+     );
+     this.get(
+       '/api/teams/gh/channels/prs/messages',
+       jsonResponse([{
+         id: 1,
+         user: {
+           name: 'Testy Testerson',
+         },
+         body: 'Hello Tests',
+        }])
+      );
+    }
+
+   module('Acceptance | forwarding routes', function(hooks) {
+     setupApplicationTest(hooks);
+
+     hooks.beforeEach(function() {
+       this.owner.register('service:auth', StubbedAuthService);
+     });
+
+     /**
+      * @type {Pretender | null}
+      */
+     let server;
+     hooks.beforeEach(function() {
+       server = new Pretender(setupServer);
+     });
+     hooks.afterEach(function() {
+       server && server.shutdown();
+       server = null;
+     });
+
+     test('forwarding from /teams', async function(assert) {
+       const auth = this.owner.lookup('service:auth');
+       auth.currentUserId = '1';
+
+       await visit('/teams');
+
+       assert.equal(currentURL(), '/teams/gh/prs');
+     });
+   });
```

In the above code that we added, apart from the actual tests, there are other functions that we used, to make the tests work correctly:

- `setupServer` - function that takes care of simulating API responses used in the acceptance tests, by intercepting http requests using [pretender](https://github.com/pretenderjs/pretender), and responding with hard coded data.
- `jsonResponse` - helper function that converts the raw response data, in this case, an array of objects, and converts it into a format that can be used by `setupServer` method.

And that's it. We have now successfully implemented conditional route forwarding, and added tests for the same.
