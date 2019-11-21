# Deleting a chat message

There's already a delete button in the `<ChatMessage>` component -- we just need to hook it up to something that does the appropriate deleting.

Let’s begin by defining a new action in [`app/components/channel-container.js`](../app/components/channel-container.js) that:

1. Makes a `DELETE` HTTP request to `/api/messages/${message.id}`
2. Throws an error if the request can't be completed for some reason
3. Removes the appropriate message from the `this.messages` array
4. Triggers a tracked property update by performing a "no-op" assignment `this.messages = this.messages`

```js
  @action async deleteMessage(message) {
    const resp = await fetch(`/api/messages/${message.id}`, {
      method: 'DELETE',
      headers: {
        'Content-Type': 'application/json',
      },
    });

    if (!resp.ok) {
      const reason = await resp.text();
      throw new Error(`Problem deleting message: ${reason}`);
    }

    const idx = this.messages
      .map(m => m.id)
      .indexOf(message.id);

    this.messages.splice(idx, 1);

    // "no-op" assignment to trigger an update
    this.messages = this.messages;
  }

```

Now we can go to [`app/components/channel-container.hbs`](../app/components/channel-container.hbs) and yield our new `deleteMessage` action:

```diff
    messages=this.messages
    actions=(hash
      createMessage=this.createMessage
+     deleteMessage=this.deleteMessage
    )
  )}}
</main>

```

In our "channel" top-level template, let’s pass `deleteMessage` to each `<ChatMessage>` component. We want `<ChatMessage>` to be as simple as possible so we’ll even fill in the `message` argument ahead of time using the `{{fn}}` helper. This is the first time we’ve seen `{{fn}}` so we’ll explain it in more detail once we’ve seen how it’s used.

Open up [`app/teams/team/channel.hbs`](../app/teams/team/channel.hbs) and make the following modifications:

```diff
    {{#each ch.messages as |message|}}
-     <ChatMessage @message={{message}} />
+     <ChatMessage
+       @message={{message}}
+       @onDelete={{fn ch.actions.deleteMessage message}}
+     />
    {{/each}}
```

In the above, we: 

1. Pass `actions.deleteMessage` to `<ChatMessage>` as `@onDelete`.
2. Fill in the first argument of `deleteMessage` using `{{fn}}`

This `{{fn}}` expression:

```hbs
@onDelete={{fn ch.actions.deleteMessage message}}
```

Is the equivalent of this in JavaScript:

```js
chatMessage.onDelete = function() { return ch.actions.deleteMessage(message) }
```

This is sometimes called “binding” or “currying”, and it’s so useful that Ember has `{{fn}}` built-in for just this purpose.

Finally, let's go into [`app/components/chat-message.hbs`](../app/components/chat-message.hbs) and hook the action up to the existing button:

```diff
  <button
    class="message__delete-button border-transparent hover:border-red-light show-on-hover hover:bg-red-lightest border-1 rounded mb-1 pl-3 pr-2 py-1"
    aria-label="permanently delete this message"
+   {{on "click" @onDelete}}
  >
```

We should now be able to hover over each message to see a "delete" button. Clicking on one of these buttons should result in the `DELETE` api call being made, and the message being removed from the screen (and our `db.json` database).

## 🙌

Great work! In this exercise we’ve seen:

1. Another example of passing actions from container to presentational components
2. How to “curry” actions so they can be easily called from anywhere in the app