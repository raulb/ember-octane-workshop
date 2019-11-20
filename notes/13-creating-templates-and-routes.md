# Creating Templates and Routes

In this exercise, we will create templates and classes for some new routes. Then, we will add our new routes to the `router.js` file. The route classes, as we saw in earlier exercises, will provide the data that the templates will display. Finally, we will wrap up this exercise by adding some tests.

Let's get started!

## Configuring the Router

The router file is where information about routes is stored. We are going to add two new child routes — `team` and `channel`. The `channel` route will be a child of the `team` route, which in turn will be a child of the `teams` (plural) route. This will give us a route hierarchy that is 3 levels deep.

Configure the routes by modifying [`app/router.js`](../app/router.js) as follows:

Replace this:

```js
this.route('teams');
```

With this:

```js
this.route('teams', function() {
  this.route('team', { path: ':teamId' }, function() {
    this.route('channel', { path: ':channelId' });
  });
});
```

## Enhancing Routes

In this section, we will be creating/editing the following route files:

- [`app/routes/teams.js`](../app/routes/teams.js)
- [`app/routes/teams/team.js`](../app/routes/teams/team.js)
- [`app/routes/teams/team/channel.js`](../app/routes/teams/team/channel.js)

In the [`teams`](../app/routes/teams.js) route, we will use the [`model`](https://api.emberjs.com/ember/3.9/classes/Route/methods/model?anchor=model) hook to return some static data to be displayed in the template. In this case, we will return an array of javascript objects. Let’s start by defining that array.

Add the following to [`app/routes/teams.js`](../app/routes/teams.js), just after the `import` statements at the top of the file.

```js
export const ALL_TEAMS = [
  {
    id: 'li',
    name: 'LinkedIn',
    order: 2,
    iconUrl:
      'https://gravatar.com/avatar/0ca1be2eaded508606982feb9fea8a2b?s=200&d=https://upload.wikimedia.org/wikipedia/commons/thumb/c/ca/LinkedIn_logo_initials.png/240px-LinkedIn_logo_initials.png',
    channels: [
      {
        id: 'general',
        name: 'general',
        description: 'LinkedIn general (professional) chat',
        teamId: 'li',
      },
      {
        id: 'secrets',
        name: 'secrets',
        description: 'professional secrets',
        teamId: 'li',
      },
    ],
  },
  {
    id: 'ms',
    name: 'Microsoft',
    order: 3,
    iconUrl:
      'https://gravatar.com/avatar/0ca1be2eaded508606982feb9fea8a2b?s=200&d=https://upload.wikimedia.org/wikipedia/commons/thumb/4/44/Microsoft_logo.svg/200px-Microsoft_logo.svg.png',
    channels: [
      {
        id: 'general',
        name: 'general',
        description: 'Microsoft general chat',
        teamId: 'ms',
      },
      {
        id: 'ie8-gripes',
        name: 'IE8 Gripes',
        description:
          'A place for whining about old browsers',
        teamId: 'ms',
      },
    ],
  },
];
```

Then add a `model` hook to the body of the class. This method returns the `ALL_TEAMS` data we just defined:

```js
model() {
  return ALL_TEAMS;
}
```

Now lets move on to defining the [`team`](../app/routes/teams/team.js) route. Create [`app/routes/teams/team.js`](../app/routes/teams/team.js) and add the following code to it:

```js
import Route from '@ember/routing/route';
import { ALL_TEAMS } from '../teams';

export default class TeamsTeamRoute extends Route {
  model({ teamId }) {
    return ALL_TEAMS.find(team => team.id === teamId);
  }
}
```

In the code shown above, the `model` hook returns an object from the `ALL_TEAMS` array that we defined earlier in [`app/routes/teams.js`](../app/routes/teams.js). The object it returns will be the one whose `id` matches the `teamId` specified in the URL for this `team` route.

Similar to how we created the `team` route, create a new `.js` file for `channel` route at [`app/routes/teams/team/channel.js`](../app/routes/teams/team/channel.js) and add the following code to it:

```js
import Route from '@ember/routing/route';

export default class TeamsTeamChannelRoute extends Route {
  model({ channelId }) {
    const team = this.modelFor('teams.team');
    const channel = team.channels.find(chan => chan.id === channelId);

    return channel;
  }
}
```

In the code shown above, the `model` hook has three steps:

1. It gets the current `team` object from its parent route (`teams.team`).
2. It searches `team.channels` for a channel whose `id` matches the `channelId` in the URL.
3. It returns the matching channel.

## Displaying Static Data in templates

We have now configured our routes to return static data from their respective `model` hooks. Now let's start creating the templates so that we can display that data.

<blockquote>
  <h3>
    💡 Mike's Tip: Where do templates live?
  </h3>
  <a href="https://github.com/mike-north">
    <img src="https://github.com/mike-north.png" height=64 align="left" style="margin-right: 10px" />
  </a>
  <p>
    Generally, in Ember apps, templates are either associated with a route or a component. Component templates live in <code>app/components</code> and route templates lives in <code>app/templates</code>.
  </p>
  <p>
    For example, the template for our <code>&lt;TeamSelector&gt;</code> component is <code>app/components/team-selector.hbs</code>. Whereas, the component for our <code>teams/team</code> route is <code>app/templates/teams/team.hbs</code> (matching the hierarchy).
  </p>
</blockquote>

In this section, we will be creating/editing the following template files:

- [`app/templates/teams.hbs`](../app/templates/teams.hbs)
- [`app/components/team-selector.hbs`](../app/components/team-selector.hbs)
- [`app/templates/teams/team.hbs`](../app/templates/teams/team.hbs)
- [`app/components/team-sidebar.hbs`](../app/components/team-sidebar.hbs)
- [`app/templates/teams/team/channel.hbs`](../app/templates/teams/team/channel.hbs)

First, refactor [`app/templates/teams.hbs`](../app/templates/teams.hbs) by replacing the existing content with a `<TeamSelector>` component. We will pass the `teams` data from the route’s model hook directly into `<TeamSelector>`.


Replace everything in the file with this:

```hbs
<TeamSelector @teams={{this.model}} />

{{outlet}}
```

Inside [`app/components/team-selector.hbs`](../app/components/team-selector.hbs), we are going to replace the existing content with code to iterate over the `@teams` array that we just passed in.

The array iteration is done using Handlebars’ [`each`](https://api.emberjs.com/ember/3.9/classes/Ember.Templates.helpers/methods/each?anchor=each) helper. For a full list of built-in Helpers, see the [`Ember.Templates.helpers`](https://api.emberjs.com/ember/release/classes/Ember.Templates.helpers/) API documentation.

Replace the contents of [`app/components/team-selector.hbs`](../app/components/team-selector.hbs) with this:

```hbs
<nav class="team-selector bg-indigo-darkest border-indigo-darkest border-r-2 pt-2 text-purple-lighter flex-none hidden sm:block">
  {{#each @teams as |team|}}
    <LinkTo
      @route="teams.team"
      @model={{team.id}}
      data-team-id={{team.id}}
      class="team-selector__team-button cursor-pointer rounded-lg p-2 pl-4 block no-underline opacity-25 opacity-100"
    >
      <div class="bg-white h-12 w-12 flex items-center justify-center text-black text-2xl font-semibold rounded-lg mb-1 overflow-hidden">
        <img
          class="team-selector__team-logo"
          src={{team.iconUrl}}
          alt={{team.name}}
        >
      </div>
    </LinkTo>
  {{/each}}
</nav>
```

Create a new template for the `team` route at `app/templates/teams/team.hbs` and add the following code:

```hbs
<TeamSidebar @team={{this.model}} />

{{outlet}}
```

In the code above, we’re using the `<TeamSidebar>` component that we already built, and passing the data from our `model` hook into it.

Now we’ll change our `<TeamSidebar>` component to dynamically display the team name.

In [`app/components/team-sidebar.hbs`](../app/components/team-sidebar.hbs), replace this static section:

```hbs
LinkedIn
```

With this:

```
{{@team.name}}
```

(`@team` is the argument we passed in.)

Then replace this static link:

```hbs
<a href="/li/general" data-channel-id="general"
  class="team-sidebar__channel-link py-1 px-4 text-white no-underline block opacity-75 bg-teal-dark">
  <span aria-hidden="true">#</span>
  general
</a>
```

With this:

```hbs
{{#each @team.channels as |channel|}}
  <LinkTo
    @route="teams.team.channel"
    @model={{channel.id}}
    @activeClass="bg-teal-dark"
    data-channel-id={{channel.id}}
    class="team-sidebar__channel-link py-1 px-4 text-white no-underline block opacity-75"
  >
    <span aria-hidden="true">#</span>
    {{channel.name}}
  </LinkTo>
{{/each}}
```

Here we’re rendering a link for each channel in `@team.channels` using [`each`](https://api.emberjs.com/ember/3.9/classes/Ember.Templates.helpers/methods/each?anchor=each) and `<LinkTo>`.

Now it’s time to add our final template for this exercise: [`app/templates/teams/team/channel.hbs`](../app/templates/teams/team/channel.hbs).

Once you’ve created the file, add the following code:

```hbs
<main class="flex-1 flex flex-col bg-white overflow-hidden channel">
  <ChannelHeader @title={{this.model.name}} @description={{this.model.description}} />

  <div class="py-4 flex-1 overflow-y-scroll channel-messages-list" role="list">
    <ChatMessage />
    <ChatMessage />
    <ChatMessage />
  </div>

  <ChannelFooter />
</main>
```

This is the code we previously removed from `app/templates/teams.hbs`. We have effectively moved it down into our child route.

## Updating Tests

We are done with the implementation for this exercise but we still need to update some of our tests.

In [`tests/acceptance/logout-test.js`](../tests/acceptance/logout-test.js) let’s modify the existing tests to reflect the new URL associated with the `team` route.

Replace this:

```js
await visit('/teams'); // Go to a URL
```

With this:

```js
await visit('/teams/li'); // Go to a URL
```

And this:

```js
assert.equal(currentURL(), '/teams'); // Make sure we've arrived
```

With this:

```js
assert.equal(currentURL(), '/teams/li'); // Make sure we've arrived
```

Now let’s turn our attention to the component tests.

The `<TeamSidebar>` component is now used with an argument, `@team`.

In [`tests/integration/components/team-sidebar-test.js`](../tests/integration/components/team-sidebar-test.js), replace this:


```js
await render(hbs`<TeamSidebar />`);
```

With this:

```js
this.set('myTeam', {
  name: 'LinkedIn',
  channels: [
    {
      name: 'general',
      id: 'general',
    },
  ],
});

await render(hbs`<TeamSidebar @team={{this.myTeam}} />`);
```

Now we can visit [`http://localhost:4200/tests?nolint`](http://localhost:4200/tests?nolint) and we should see everything passing.

## 🙌

Nice work — we have successfully restructured our app with dynamic URLs and nested routes.