# Logux Docs

<img align="right" width="95" height="148" title="Logux logotype"
     src="https://logux.io/branding/logotype.svg">

Logux is a new way to connect client (webapp, mobile app) and server. Instead of sending HTTP requests (e.g., AJAX, REST, and GraphQL) it synchronizes log of operations between client, server, and other clients through WebSocket.

It was created on top of **[CRDT]**, with ideas of having live updates and optimistic UI, and being offline-first by design.

* Built-in **optimistic UI** will improve UI performance.
* Built-in **live updates** allow to create collaborative tools (like Google Docs).
* Built-in **offline-first** principle respect will improve UX on unstable connection. It is useful both for the next billion users and New York subway.
* Compatible with modern stack: **Redux** API, works with **any back-end language** and **any database**.

Read more: [logux.io]<br>
Ask your questions at [our Gitter]<br>
Commercial support: [`logux@evilmartians.com`]

<a href="https://evilmartians.com/?utm_source=logux-docs">
  <img src="https://evilmartians.com/badges/sponsored-by-evil-martians.svg"
       alt="Sponsored by Evil Martians" width="236" height="54">
</a>

[`logux@evilmartians.com`]: mailto:logux@evilmartians.com
[our Gitter]: https://gitter.im/logux/logux
[logux.io]: https://logux.io/
[CRDT]: https://slides.com/ai/crdt


## Client Example

<details open><summary>React/Redux client</summary>

Using [`@logux/redux`](https://github.com/logux/redux/):

```js
export const Counter = () => {
  // Will load current counter from server and subscribe to counter changes
  const isSubscribing = useSubscription(['counter'])
  if (isSubscribing) {
    return <Loader />
  } else {
    const counter = useSelector(state => state.counter)
    const dispatch = useDispatch()
    return <>
      <div>{ counter }</div>
      // `dispatch.sync()` instead of Redux `dispatch()` will send action to all clients
      <button onClick={ dispatch.sync({ type: 'INC' }) }>
    </>
  }
}
```

</details>
<details><summary>Pure JS client</summary>

Using [`@logux/client`](https://github.com/logux/client/):

```js
log.on('add', (action, meta) => {
  if (action.type === 'INC') {
    counter.innerHTML = parseInt(counter.innerHTML) + 1
  }
})

increase.addEventListener('click', () => {
  log.add({ type: 'INC' }, { sync: true })
})

log.add({ type: 'logux/subscribe' channel: 'counter' }, { sync: true })
```

</details>


## Server Example

<details open><summary>Node.js</summary>

Using [`@logux/server`](https://github.com/logux/server/):

```js
server.channel('counter', {
  access () {
    // Access control is mandatory. API was designed to make it harder to write dangerous code.
    return true
  },
  async init (ctx) {
    // Load initial state when client subscribing to the channel.
    // You can use any database.
    let value = await db.get('counter')
    ctx.sendBack({ type: 'INC', value })
  }
})

server.type('INC', {
  resend () {
    return { channel: 'counter' }
  },
  access () {
    return true
  },
  async process () {
    // Don’t forget to keep action atomic
    await db.set('counter', 'value += 1')
  }
})
```

</details>
<details><summary>Ruby on Rails</summary>

Using [`logux_rails`](https://github.com/logux/logux_rails/):

```ruby
# app/logux/channels/counter.rb
module Channels
  class Counter < Logux::ChannelController
    def initial_data
      [{ type: 'INC', value: db.counter }]
    end
  end
end
```

```ruby
# app/logux/actions/inc.rb
module Actions
  class Inc < Logux::ActionController
    def inc
      # Don’t forget to keep action atomic
      db.update_counter! 'value += 1'
    end
  end
end
```

```ruby
# app/logux/policies/channels/counter.rb
module Policies
  module Channels
    class Counter < Policies::Base
      # Access control is mandatory. API was designed to make it harder to write dangerous code.
      def subscribe?
        true
      end
    end
  end
end
```

```ruby
# app/logux/policies/actions/inc.rb
module Policies
  module Actions
    class inc < Policies::Base
      def inc?
        true
      end
    end
  end
end
```

</details>
<details><summary>Any other HTTP server</summary>

You can use any HTTP server with Logux WebSocket proxy server. Here is a PHP-like pseudocode example:

```php
<?php
$req = json_decode(file_get_contents('php://input'), true);
if ($req['password'] == LOGUX_PASSWORD) {
  foreach ($req['commands'] as $command) {
    if ($command[0] == 'action') {
      $action = $command[1];
      $meta = $command[2];

      if ($action['type'] == 'logux/subscribe') {
        echo '[["approved"],';
        $value = $db->getCounter();
        send_json_http_post(LOGUX_HOST, [
          'password' => LOGUX_PASSWORD,
          'version' => 1,
          'commands' => [
            [
              'action',
              ['type' => 'INC', 'value' => $value],
              ['clients' => get_client_id($meta['id'])]
            ]
          ]
        ]);
        echo '["processed"]]';

      } elseif ($action['type'] == 'inc') {
        $db->updateCounter('value += 1');
        echo '[["approved"],["processed"]]';
      }
    }
  }
}
```

</details>
