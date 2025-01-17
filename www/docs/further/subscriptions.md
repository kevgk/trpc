---
id: subscriptions
title: Subscriptions / WebSockets
sidebar_label: Subscriptions / WebSockets
slug: /subscriptions
---

:::info
Subscriptions & WebSockets are in beta, alpha & might change without a major version bump. However, feel free to use them and report any issue you may find on [GitHub](https://github.com/trpc/trpc)
:::

## Using Subscriptions

:::tip
- For a full-stack example have a look at [/examples/next-prisma-starter-websockets](https://github.com/trpc/trpc/tree/main/examples/next-prisma-starter-websockets).
- For a bare-minumum Node.js example see [/examples/standalone-server](https://github.com/trpc/trpc/tree/main/examples/standalone-server).

:::

### Adding a subscription procedure


```tsx
import * as trpc from '@trpc/server';
import { EventEmitter } from 'events';

// create a global event emitter (could be replaced by redis, etc)
const ee = new EventEmitter()

trpc.router()
  .subscription('onAdd', {
    resolve({ ctx }) {
      // `resolve()` is triggered for each client when they start subscribing `onAdd`

      // return a `Subscription` with a callback which is triggered immediately
      return new trpc.Subscription<Post>((emit) => {
        const onAdd = (data: Post) => {
          // emit data to client
          emit.data(data)
        };

        // trigger `onAdd()` when `add` is triggered in our event emitter
        ee.on('add', onAdd);

        // unsubscribe function when client disconnects or stops subscribing
        return () => {
          ee.off('add', onAdd);
        };
      });
    },
  })
  .mutation('add', {
    input: z.object({
      id: z.string().uuid().optional(),
      text: z.string().min(1),
    }),
    async resolve({ ctx, input }) {
      const post = { ...input } /* [..] add to db */

      ee.emit('add', post);
      return post;
    },
  })
```

### Creating a WebSocket-server

```bash
yarn add ws
```

```ts
import ws from 'ws';
import { applyWSSHandler } from '@trpc/server/adapters/ws';
import { appRouter } from './routers/app';
import { createContext } from './trpc';
const wss = new ws.Server({
  port: 3001,
});
const handler = applyWSSHandler({ wss, router: appRouter, createContext });

wss.on('connection', (ws) => {
  console.log(`➕➕ Connection (${wss.clients.size})`);
  ws.once('close', () => {
    console.log(`➖➖ Connection (${wss.clients.size})`);
  });
});
console.log('✅ WebSocket Server listening on ws://localhost:3001');

process.on('SIGTERM', () => {
  console.log('SIGTERM');
  handler.broadcastReconnectNotification();
  wss.close();
});
```

### Setting `TRPCClient` to use WebSockets

:::tip
You can [use Links](../client/links.md) to route queries and/or mutations to HTTP transport and subscriptions over WebSockets.
:::

```tsx
import { createWSClient, wsLink } from '@trpc/client/links/wsLink';
import { httpBatchLink } from '@trpc/client/links/httpBatchLink';

// create persistent WebSocket connection
const wsClient = createWSClient({
  url: `ws://localhost:3001`,
});

// configure TRPCClient to use WebSockets transport
const client = createTRPCClient<AppRouter>({
  links: [
    wsLink({
      client: wsClient,
    }),
  ],
});
```




### Using React

See [/examples/next-prisma-starter-websockets](https://github.com/trpc/trpc/tree/main/examples/next-prisma-starter-websockets).

## WebSockets RPC Specification

> You can read more details by drilling into the TypeScript definitions: 
>
>- [/packages/server/src/rpc/envelopes.ts](https://github.com/trpc/trpc/tree/main/packages/server/src/rpc/envelopes.ts)
>- [/packages/server/src/rpc/codes.ts](https://github.com/trpc/trpc/tree/main/packages/server/src/rpc/codes.ts).


### `query` / `mutation`


#### Request

```ts
{
  id: number | string;
  jsonrpc?: '2.0';
  method: 'query' | 'mutation';
  params: {
    path: string;
    input?: unknown; // <-- pass input of procedure, serialized by transformer
  };
}
```

#### Response

_... below, or an error._

```ts
{
  id: number | string;
  jsonrpc: '2.0';
  result: {
    type: 'data'; // always 'data' for mutation / queries
    data: TOutput; // output from procedure
  };
}
```


### `subscription` / `subscription.stop`


#### Start a subscription

```ts
{
  id: number | string;
  jsonrpc?: '2.0';
  method: 'subscription';
  params: {
    path: string;
    input?: unknown; // <-- pass input of procedure, serialized by transformer
  };
}
```

#### To cancel a subscription, call `subscription.stop`

```ts
{
  id: number | string; // <-- id of your created subscription
  jsonrpc?: '2.0';
  method: 'subscription.stop';
}
```

#### Subscription response shape

_... below, or an error._

```ts
{
  id: number | string;
  jsonrpc: '2.0';
  result: (
    | {
      type: 'data';
        data: TData; // subscription emitted data
      }
    | {
        type: 'started'; // sub started
      }
    | {
        type: 'stopped'; // sub stopped
      }
  )
}
```

## Errors

See https://www.jsonrpc.org/specification#error_object or [Error Formatting](../server/error-formatting.md).


## Notifications from Server to Client


### `{id: null, type: 'reconnect' }`

Tells clients to reconnect before shutting down server. Invoked by `wssHandler.broadcastReconnectNotification()`.
