# Creating new chat messages

In this step, we'll continue to explore the idea of container and presentational components by passing a `createMessage` action from the `<ChannelContainer />` into a presentational component. All the presentational component will do is invoke the action, without any concern for what the particulars of "saving a message" might entail.

Let's turn our attention to the [`app/components/channel-container.js`](../app/components/channel-container.js) file.

Let's begin by injecting the `auth` service, since we will need it in order to obtain the userId of the currently logged-in user.

```js
  /**
   * @type {AuthService}
   */
  @service auth;
```

Next, let's enhance our channel container by implementing a `createMessage` action. This should...

1. take a chat message body (a string) as an argument
2. make a `POST` API call with `Content-Type: application/json` header to the `/api/messages` endpoint, with a payload like `{ channelId: 'foo', teamId: 'bar', body: 'hello channel', userId: 123 }`
3. throw an error if the HTTP response cannot be completed, or if the status code looks non-successful
4. add the new message that the server returns to the `this.messages` array

```js

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
      throw new Error(
        'Problem creating message: ' + (await resp.text())
      );
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

In this component's template, let's create a new `actions` object that's yielded out, and pass our new action along as a property. Consumers can then do something like `channel.actions.createMessage` to access this function. Make the following change to [`app/components/channel-container.hbs`](../app/components/channel-container.hbs)

```diff
<main class="flex-1 flex flex-col bg-white overflow-hidden channel"
  {{did-insert this.loadMessages}}
  {{did-update this.loadMessages @channel}}
>
  {{yield (hash
    messages=this.messages
+   actions=(hash
+     createMessage=this.createMessage
+   )
  )}}
</main>
```

Consume this new action that's yielded out of the component in [`app/templates/teams/team/channel.hbs`](../app/templates/teams/team/channel.hbs)

```diff
    {{/each}}
  </div>

-  <ChannelFooter />
+  <ChannelFooter @createMessage={{ch.actions.createMessage}} />
</ChannelContainer>
```

We now have a function available to `<ChannelFooter />`, either as `@createMessage` in the component's `.hbs` file or `this.args.createMessage` within the `.js` file. When we invoke `createMessage` it will create a new chat message for the current user, in the current channel.

Before we use it, let's stop and think about some other reasonable behavior we might want in this component

- The user should be able to "click" the "send" button to create the message, or press `Cmd + Enter`
  - this is an indication that we probably want the "submitting" to happen via `<form>` and the `onsubmit` event.
- The "send" button should be disabled unless the user actually types something in the message field
  - this is an indication that we need to keep track of the `<input>`'s value at all times, and create some derived state (`isDisabled`) based on it

Let's take care of the "disable send if the message is blank" functionality first.

Generate a component class for `<ChannelFooter>`:

```
$ ember g component-class channel-footer
```

Now replace the contents of [`app/components/channel-footer.js`](`../app/components/channel-footer.js`)
with this:

```js
import Component from '@glimmer/component';
import { action } from '@ember/object';
import { tracked } from '@glimmer/tracking';

export default class ChannelFooterComponent extends Component {
  @tracked messageBody; // the state of the `<input>` value

  @action
  updateMessageBody(evt) {
    // action fired on `<input>`'s "input" event
    this.messageBody = evt.target.value; // updating our state
  }

  // derived state: whether messageBody is empty or not
  get isDisabled() {
    return !this.messageBody;
  }
}
```

We can hook this up with a few changes to [`app/components/channel-footer.hbs`](`../app/components/channel-footer.hbs`):

```diff
    <input id="message-input" class="channel-footer__message-input w-full px-4"
      placeholder="Message #general" type="text"
+     value={{this.messageBody}}
+     {{on "input" this.updateMessageBody}}
    >

-   <button disabled
+   <button disabled={{this.isDisabled}}
-     class="channel-footer__message-send-button font-bold uppercase opacity-50 bg-grey-dark text-white border-teal-dark p-2">
+     class="
+       channel-footer__message-send-button font-bold uppercase text-white border-teal-dark p-2
+       {{if this.isDisabled "bg-grey-dark opacity-50" "bg-teal-dark"}}
+     "
+   >
      SEND
    </button>
  </form>
```

Now let's hook up that submit event. Make one more change to [`app/components/channel-footer.hbs`](`../app/components/channel-footer.hbs`), to use the `{{on}}` modifier to fire an `this.onSubmit` action whenever the `<form>` fires its `"submit"` event.

```diff
<!-- Channel Footer -->
<footer class="pb-6 px-4 flex-none channel-footer">
- <form class="flex w-full rounded-lg border-2 border-grey overflow-hidden" aria-labelledby="message-label">
+ <form class="flex w-full rounded-lg border-2 border-grey overflow-hidden" aria-labelledby="message-label"
+   {{on "submit" this.onSubmit}}
+ >
    <h1 id="message-label" class="sr-only">
      Message Input
    </h1>
```

Go back to [`app/components/channel-footer.js`](`../app/components/channel-footer.js`) and add the appropriate action

```js
  @action
  async onSubmit(evt) {
    evt.preventDefault();

    await this.args.createMessage(this.messageBody); // call the fn we were passed as an arg

    if (this.isDestroyed || this.isDestroying) return;

    this.messageBody = ''; // reset the message field
  }
```

## 🙌

Fantastic! Give the app a try. We should now be able to create chat messages!
