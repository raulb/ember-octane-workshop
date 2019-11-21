# Singleton-Driven UI - Notification Center

While container components are a great way to gather together related data and behavior, sometimes it's more convenient to use a service to expose stuff like this as a ”singleton” — a single object that’s shared across the whole app. This is particularly true when many parts of an app want to to get involved with some data or behavior. A great example of this kind of pattern is a notification center.

## The Notification Component

Let's begin by generating a `<Notification>` component:

```sh
ember g component notification
```

Now replace the contents of [`app/components/notification.hbs`](../app/components/notification.hbs) with this:

```hbs
<div
  class="
    notification flex flex-row items-center bg-{{@notification.color}}
    text-white text-sm font-bold px-4 py-3 notification-transition
    {{if @notification.entering "entering" ""}}
    {{if @notification.leaving "leaving" ""}}
  "
  role="alert"
>
  {{#if hasBlock}}
    {{yield}}
  {{else}}
    <svg class="fill-current w-4 h-4 mr-2" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20">
      <path d="M12.432 0c1.34 0 2.01.912 2.01 1.957 0 1.305-1.164 2.512-2.679 2.512-1.269 0-2.009-.75-1.974-1.99C9.789 1.436 10.67 0 12.432 0zM8.309 20c-1.058 0-1.833-.652-1.093-3.524l1.214-5.092c.211-.814.246-1.141 0-1.141-.317 0-1.689.562-2.502 1.117l-.528-.88c2.572-2.186 5.531-3.467 6.801-3.467 1.057 0 1.233 1.273.705 3.23l-1.391 5.352c-.246.945-.141 1.271.106 1.271.317 0 1.357-.392 2.379-1.207l.6.814C12.098 19.02 9.365 20 8.309 20z" />
    </svg>
    <p>{{@notification.body}}</p>
  {{/if}}
</div>
```

This component renders default HTML when used in “inline form”:

```hbs
<Notification @notification={{someNotification}} />
```

But it also allows us to use custom HTML in "block form":

```hbs
<Notification @notification={{someNotification}}>
  Something custom
</Notification>
```

In the template above, we can see `{{#if hasBlock}}` -- this is what allows us to conditionally render the SVG _only_ when inline form is used.

## The Service

Next, let's generate a service to hold the collection of notifications:

```sh
ember g service notifications
```

and implement [`app/services/notifications.js`](../app/services/notifications.js) as follows

```js
import Service from '@ember/service';
import { action } from '@ember/object';
import { tracked } from '@glimmer/tracking';

// Generates a random ID
const getId = () => Math.round(Math.random() * 10000000);

const HANG_TIME = 3000; // ms

export default class NotificationsService extends Service {
  @tracked
  messages = [];

  @action
  notify(body, color = 'indigo-darker') {
    const id = getId();
    const newNotification = { id, body, color };

    this.messages = [...this.messages, newNotification];

    // Remove after HANG_TIME is complete
    setTimeout(() => {
      this.messages = this.messages.filter(m => m !== newNotification);
    }, HANG_TIME);
  }
}
```

Now, anywhere we need to be able to provide notifications, we need only inject this service and call `notifications.notify('Message');`.

## The List of Notifications

We'll need a component for the list of notifications. Let's generate one:

```sh
ember generate component notification-list --with-component-class
```

Now implement [`app/components/notification-list.js`](../app/components/notification-list.js) as:

```js
import Component from '@glimmer/component';
import { inject as service } from '@ember/service';

export default class NotificationListComponent extends Component {
  @service notifications;
}
```

And [`app/components/notification-list.hbs`](../app/components/notification-list.hbs) as:

```hbs
<div class="notifications-container z-10">
  {{#each this.notifications.messages as |msg|}}
    {{#if (eq msg.color "green-dark")}}
      <Notification @notification={{msg}}>
        <img src="https://media.giphy.com/media/toBi2rizjV8CswqIXG/giphy.gif" width="140" class="mr-20">
        {{msg.body}}
      </Notification>
    {{else}}
      <Notification @notification={{msg}} />
    {{/if}}
  {{/each}}
</div>
```

Let's use this component at the top of the channel, above all of the messages.

Open [`app/templates/teams/team/channel.hbs`](../app/templates/teams/team/channel.hbs) and make the following change:

```diff
  <ChannelHeader @title={{this.model.name}} @description={{this.model.description}} />
+ <NotificationList />
  <div class="py-4 flex-1 overflow-y-scroll channel-messages-list" role="list">
```

## Notifying on Important Events

Now all we have left to do is use the service to create notifications in response to important events.

First, inject our new `notifications` service onto the `<ChannelContainer>` component:

```diff
  @service auth;
+ @service notifications;
```

Then replace the error-handling section of `deleteMessage`:

```js
if (!resp.ok) {
  const reason = await resp.text();
  throw new Error(`Problem deleting message: ${reason}`);
}
```

With this:

```js
if (!resp.ok) {
  const reason = await resp.text();
  this.notifications.notify(`Problem deleting message: ${reason}`, 'red');
  return;
}
```

And finally add a success notification if we made it all the way to the end of `deleteMessage`:

```js
this.notifications.notify('Problem deleting message', 'red');
```

Then update `createMessage` in a similar manner:

```diff
  @action
  async createMessage(body) {
    // Grab `channelId` and `teamId` from `this.args`
    const {
      channel: { id: channelId, teamId },
    } = this.args;

    // Make a POST request to /api/message with:
    // - channelId
    // - teamId
    // - body
    // - the current userId
    const resp = await fetch(`/api/messages`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        channelId,
        teamId,
        body,
        userId: this.auth.currentUserId,
      }),
    });

    // Why’s this important? Let’s discuss as a group...
    if (this.isDestroyed || this.isDestroying) return;

    // If the response was *not OK* then throw an error
    if (!resp.ok) {
      const reason = await resp.text();
-     throw new Error(`Problem creating message: ${reason}`);
+     this.notifications.notify(`Problem creating message: ${reason}`, 'red');
+   } else {
+     this.notifications.notify('Created new message', 'green-dark');
    }

    // Grab the message data from the response
    const newMessage = await resp.json();

    // Another one of these...
    if (this.isDestroyed || this.isDestroying) return;

    // Set `this.messages` to a new array with our brand new message at the end.
    // We’ll also sneak in an extra `user` property while we’re here for some reason.
    this.messages = [
      ...this.messages,
      { ...newMessage, user: this.auth.currentUser },
    ];

    return newMessage;
  }
```

You should now see notifications upon creation and deletion of messages, both for successful completion of these operations and when the app encounters an error.
