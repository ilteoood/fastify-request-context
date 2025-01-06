# @fastify/request-context

[![NPM Version][npm-image]][npm-url]
[![NPM Downloads][downloads-image]][downloads-url]
[![CI](https://github.com/fastify/fastify-request-context/actions/workflows/ci.yml/badge.svg?branch=master)](https://github.com/fastify/fastify-request-context/actions/workflows/ci.yml)

Request-scoped storage support, based on [AsyncLocalStorage](https://nodejs.org/api/async_context.html#asynchronous-context-tracking)

Inspired by work done in [@fastify/http-context](https://github.com/thorough-developer/fastify-http-context).

This plugin introduces thread-local request-scoped http context, where any variables set within the scope of a single http call won't be overwritten by simultaneous calls to the api
nor will variables remain available once a request is completed.

Frequent use-cases are persisting request-aware logger instances and user authorization information.

## Getting started

First install the package:

```bash
npm i @fastify/request-context
```

Next, set up the plugin:

```js
const { fastifyRequestContext } = require('@fastify/request-context')
const fastify = require('fastify');

fastify.register(fastifyRequestContext);
```

Or customize hook and default store values:

```js
const { fastifyRequestContext } = require('@fastify/request-context')
const fastify = require('fastify');

fastify.register(fastifyRequestContext, {
  hook: 'preValidation',
  defaultStoreValues: {
    user: { id: 'system' }
  }
});
```

Default store values can be set through a function as well:

```js
const { fastifyRequestContext } = require('@fastify/request-context')
const fastify = require('fastify');

fastify.register(fastifyRequestContext, {
  defaultStoreValues: request => ({
    log: request.log.child({ foo: 123 })
  })
});
```

This plugin accepts options `hook` and `defaultStoreValues`, `createAsyncResource`.

* `hook` allows you to specify to which lifecycle hook should request context initialization be bound. Note that you need to initialize it on the earliest lifecycle stage that you intend to use it in, or earlier. Default value is `onRequest`.
* `defaultStoreValues` / `defaultStoreValues(req: FastifyRequest)` sets initial values for the store (that can be later overwritten during request execution if needed). Can be set to either an object or a function that returns an object. The function will be sent the request object for the new context. This is an optional parameter.
* `createAsyncResource` can specify a factory function that creates an extended `AsyncResource` object.

From there you can set a context in another hook, route, or method that is within scope.

Request context (with methods `get` and `set`) is exposed by library itself, but is also available as decorator on `fastify.requestContext` app instance as well as on `req` request instance.

For instance:

```js
const { fastifyRequestContext, requestContext } = require('@fastify/request-context')
const fastify = require('fastify');

const app = fastify({ logger: true })
app.register(fastifyRequestContext, {
  defaultStoreValues: {
    user: { id: 'system' }
  },
  createAsyncResource: (req, context) => new MyCustomAsyncResource('custom-resource-type', req.id, context.user.id)
});

app.addHook('onRequest', (req, reply, done) => {
  // Overwrite the defaults.
  // This is completely equivalent to using app.requestContext or just requestContext
  req.requestContext.set('user', { id: 'helloUser' });
  done();
});

// this should now get `helloUser` instead of the default `system`
app.get('/', (req, reply) => {
  // requestContext singleton exposed by the library retains same request-scoped values that were set using `req.requestContext`
  const user = requestContext.get('user');
  reply.code(200).send( { user });
});

app.get('/decorator', function (req, reply) {
  // requestContext singleton exposed as decorator in the fastify instance and can be retrieved:
  const user = this.requestContext.get('user'); // using `this` thanks to the handler function binding
  const theSameUser = app.requestContext.get('user'); // directly using the `app` instance
  reply.code(200).send( { user });
});

app.listen({ port: 3000 }, (err, address) => {
  if (err) throw err
  app.log.info(`server listening on ${address}`)
});

return app.ready()
```

## TypeScript

In TypeScript you are expected to augment the module to type your context:

```ts
import {requestContext} from '@fastify/request-context'

declare module '@fastify/request-context' {
  interface RequestContextData {
    foo: string
  }
}

// Type is "string" (if "strictNullChecks: true" in your tsconfig it will be "string | undefined")
const foo = requestContext.get('foo')
// Causes a type violation as 'bar' is not a key on RequestContextData
const bar = requestContext.get('bar')
```

If you have `"strictNullChecks": true` (or have `"strict": true`, which sets `"strictNullChecks": true`) in your TypeScript configuration, you will notice that the type of the returned value can still be `undefined` even though the `RequestContextData` interface has a specific type. For a discussion about how to work around this and the pros/cons of doing so, please read [this issue (#93)](https://github.com/fastify/fastify-request-context/issues/93).

[npm-image]: https://img.shields.io/npm/v/@fastify/request-context.svg
[npm-url]: https://npmjs.org/package/@fastify/request-context
[downloads-image]: https://img.shields.io/npm/dm/fastify-request-context.svg
[downloads-url]: https://npmjs.org/package/@fastify/request-context
