[![npm version](https://badge.fury.io/js/mappersmith.svg)](http://badge.fury.io/js/mappersmith)
[![Build Status](https://travis-ci.org/tulios/mappersmith.svg?branch=master)](https://travis-ci.org/tulios/mappersmith)
# Mappersmith

__Mappersmith__ is a lightweight rest client for node.js and the browser. It creates a client for your API, gathering all configurations into a single place, freeing your code from HTTP configurations.

## Table of Contents

1. [Installation](#installation)
1. [Usage](#usage)
  1. [Commonjs](#commonjs)
  1. [Configuring my resources](#resource-configuration)
    1. [Parameters](#parameters)
    1. [Default parameters](#default-parameters)
    1. [Body](#body)
    1. [Headers](#headers)
    1. [Alternative host](#alternative-host)
  1. [Promises](#promises)
  1. [Response object](#response-object)
  1. [Middlewares](#middlewares)
  1. [Testing Mappersmith](#testing-mappersmith)
  1. [Gateways](#gateways)
1. [Development](#development)

## <a name="installation"></a> Installation

#### NPM

```sh
npm install mappersmith --save
# yarn add mappersmith
```

#### Browser

Download the tag/latest version from the dist folder.

#### Build from the source

Install the dependencies

```sh
yarn
```

Build

```sh
npm run build
npm run release # for minified version
```

## <a name="usage"></a> Usage

To create a client for your API you will need to provide a simple manifest. If your API reside in the same domain as your app you can skip the `host` configuration. Each resource has a name and a list of methods with its definitions, like:

```javascript
// 1) Import
import forge from 'mappersmith'

// 2) Forge your client with your API manifest
const github = forge({
  host: 'https://status.github.com',
  resources: {
    Status: {
      current: { path: '/api/status.json' },
      messages: { path: '/api/messages.json' },
      lastMessage: { path: '/api/last-message.json' },
    }
  }
})

// profit!
github.Status.lastMessage().then((response) => {
  console.log(`status: ${response.data()}`)
})
```

## <a name="commonjs"></a> Commonjs

If you are using _commonjs_, your `require` should look like:

```javascript
const forge = require('mappersmith').default
```

## <a name="resource-configuration"></a> Configuring my resources

Each resource has a name and a list of methods with its definitions. A method definition can have host, path, method, headers, params, bodyAttr and headersAttr. Example:

```javascript
const client = forge({
  resources: {
    User: {
      all: { path: '/users' }

      // {id} is a dynamic segment and will be replaced by the parameter "id"
      // when called
      byId: { path: '/users/{id}' },

      // {group} is also a dynamic segment but it has default value "general"
      byGroup: { path: '/users/groups/{group}', params: { group: 'general' } },
    },
    Blog: {
      // The HTTP method can be configured through the `method` key, and a default
      // header "X-Special-Header" has been configured for this resource
      create: { method: 'post', path: '/blogs', headers: { 'X-Special-Header': 'value' } },

      // There are no restrictions for dynamic segments and HTTP methods
      addComment: { method: 'put', path: '/blogs/{id}/comment' }
    }
  }
})
```

### <a name="parameters"></a> Parameters

If your method doesn't require any parameter, you can just call it without them:

```javascript
client.User.all() // https://my.api.com/users
```

Every parameter that doesn't match a pattern `{parameter-name}` in path will be sent as part of the query string:

```javascript
client.User.all({ active: true }) // https://my.api.com/users?active=true
```

When a method requires a parameters and the method is called without it, __Mappersmith__ will raise an error:

```javascript
client.User.byId(/* missing id */)
// throw '[Mappersmith] required parameter missing (id), "/users/{id}" cannot be resolved'
```

### <a name="default-parameters"></a> Default Parameters

It is possible to configure default parameters for your resources, just use the key `params` in the definition. It will replace params in the URL or include query strings.

If we call `client.User.byGroup` without any params it will default `group` to "general"

```javascript
client.User.byGroup() // https://my.api.com/users/groups/general
```

And, of course, we can override the defaults:

```javascript
client.User.byGroup({ group: 'cool' }) // https://my.api.com/users/groups/cool
```

### <a name="body"></a> Body

To send values in the request body (usually for POST, PUT or PATCH methods) you will use the special parameter `body`:

```javascript
client.Blog.create({
  body: {
    title: 'Title',
    tags: ['party', 'launch']
  }
})
```

By default, it will create a _urlencoded_ version of the object (`title=Title&tags[]=party&tags[]=launch`). If the body used is not an object it will use the original value. If `body` is not possible as a special parameter for your API you can configure it through the param `bodyAttr`:

```javascript
// ...
{
  create: { method: 'post', path: '/blogs', bodyAttr: 'payload' }
}
// ...

client.Blog.create({
  payload: {
    title: 'Title',
    tags: ['party', 'launch']
  }
})
```

__NOTE__: It's possible to post body as JSON, check the `EncodeJsonMiddleware` bellow for more information

### <a name="headers"></a> Headers

To define headers in the method call use the parameter `headers`:

```javascript
client.User.all({ headers: { Authorization: 'token 1d1435k'} })
```

If `headers` is not possible as a special parameter for your API you can configure it through the param `headersAttr`:

```javascript
// ...
{
  all: { path: '/users', headersAttr: 'h' }
}
// ...

client.User.all({ h: { Authorization: 'token 1d1435k'} })
```

### <a name="alternative-host"></a> Alternative host

There are some cases where a resource method resides in another host, in those cases you can use the `host` key to configure a new host:

```javascript
// ...
{
  all: { path: '/users', host: 'http://old-api.com' }
}
// ...

client.User.all() // http://old-api.com/users
```

## <a name="promises"></a> Promises

__Mappersmith__ does not apply any polyfills, it depends on a native Promise implementation to be supported. If your environment doesn't support Promises, please apply the polyfill first. One option can be [then/promises](https://github.com/then/promise)

In some cases is not possible to use/assign the global `Promise` constant, for those cases you can define the promise implementation used by Mappersmith.

For example, using the project [rsvp.js](https://github.com/tildeio/rsvp.js/) (a tiny implementation of Promises/A+):

```javascript
import RSVP from 'rsvp'
import { configs } from 'mappersmith'

configs.Promise = RSVP.Promise
```

All `Promise` references in Mappersmith use `configs.Promise`. The default value is the global Promise.

## <a name="response-object"></a> Response object

Mappersmith will provide an instance of its own `Response` object to the promises. This object has the methods:

* `request()` - Returns the original [Request](https://github.com/tulios/mappersmith/blob/master/src/request.js)
* `status()` - Returns the status number
* `success()` - Returns true for status greater than 200 and lower than 400
* `headers()` - Returns an object with all headers, keys in lower case
* `data()` - Returns the response data, if `Content-Type` is `application/json` it parses the response and returns an object

## <a name="middlewares"></a> Middlewares

The behavior between your client and the API can be customized with middlewares. A middleware is a function which returns an object with two methods: request and response.

The `request` method receives an instance of [Request](https://github.com/tulios/mappersmith/blob/master/src/request.js) object and it must return a Request. The method `enhance` can be used to generate a new request based on the previous one.

The `response` method receives a function which returns a `Promise` resolving the [Response](https://github.com/tulios/mappersmith/blob/master/src/response.js). This function must return a `Promise` resolving the Response. The method `enhance` can be used to genenrate a new response based on the previous one.

You don't need to implement both methods, you can define only the phase you need.

Example:

```javascript
const MyMiddleware = () => ({
  request(request) {
    return request.enhance({
      headers: { 'x-special-request': '->' }
    })
  },

  response(next) {
    return next().then((response) => response.enhance({
      headers: { 'x-special-response': '<-' }
    }))
  }
})
```

The middleware can be configured using the key `middlewares` in the manifest, example:

```javascript
const client = forge({
  middlewares: [ MyMiddleware ],
  resources: {
    User: {
      all: { path: '/users' }
    }
  }
})
```

### Built-in middlewares

#### EncodeJson

Automatically encode your objects into JSON

```javascript
import EncodeJson from 'mappersmith/middlewares/encode-json'

const client = forge({
  middlewares: [ EncodeJson ],
  /* ... */
})

client.User.all({ body: { name: 'bob' } })
// => body: {"name":"bob"}
// => header: "Content-Type=application/json;charset=utf-8"
```

#### GlobalErrorHandler

Provides a catch-all function for all requests. If the catch-all function returns `true` it prevents the original promise to continue.

```javascript
import GlobalErrorHandler, { setErrorHandler } from 'mappersmith/middlewares/global-error-handler'

setErrorHandler((response) => {
  console.log('global error handler')
  return response.status() === 500
})

const client = forge({
  middlewares: [ GlobalErrorHandler ],
  /* ... */
})

client.User
  .all()
  .catch((response) => console.error('my error'))

// If status != 500
// output:
//   -> global error handler
//   -> my error

// IF status == 500
// output:
//   -> global error handler
```

#### Log

Log all requests and responses. Might be useful in development mode.

```javascript
import Log from 'mappersmith/middlewares/log'

const client = forge({
  middlewares: [ Log ],
  /* ... */
})
```

## <a name="testing-mappersmith"></a> Testing Mappersmith

Mappersmith plays nice with all test frameworks, the generated client is a plain javascript object and all the methods can be mocked without any problem. However, this experience can be greatly improved with the test library.

The test library has 4 utilities: `install`, `uninstall`, `mockClient` and `mockRequest`

#### install and uninstall

They are used to setup the test library, __example using jasmine__:

```javascript
import { install, uninstall } from 'mappersmith/test'

describe('Feature', () => {
  beforeEach(() => install())
  afterEach(() => uninstall())
})
```

#### mockClient

`mockClient` offers a high level abstraction, it works directly on your client mocking the resources and their methods.

It accepts the methods:

* `resource(resourceName)`, ex: `resource('Users')`
* `method(resourceMethodName)`, ex: `method('byId')`
* `with(resourceMethodArguments)`, ex: `with({ id: 1 })`
* `status(statusNumber)`, ex: `status(204)`
* `response(responseData)`, ex: `response({ user: { id: 1 } })`

Example using __jasmine__:

```javascript
import forge from 'mappersmith'
import { install, uninstall, mockClient } from 'mappersmith/test'

describe('Feature', () => {
  beforeEach(() => install())
  afterEach(() => uninstall())

  it('works', (done) => {
    const myManifest = {} // Let's assume I have my manifest here
    const client = forge(myManifest)

    mockClient(client)
      .resource('User')
      .method('all')
      .response({ allUsers: [{id: 1}] })

    // now if I call my resource method, it should return my mock response
    client.User
      .all()
      .then((response) => expect(response.data()).toEqual({ allUsers: [{id: 1}] }))
      .then(done)
  })
})
```

To mock a failure just use the correct HTTP status, example:

```javascript
// ...
mockClient(client)
  .resource('User')
  .method('byId')
  .with({ id: 'ABC' })
  .status(422)
  .response({ error: 'invalid ID' })
// ...
```

The method `with` accepts the body and headers attributes, example:

```javascript
// ...
mockClient(client)
  .with({
    id: 'abc',
    headers: { 'x-special': 'value'},
    body: { payload: 1 }
  })
  // ...
```

#### mockRequest

`mockRequest` offers a low level abstraction, very useful for automations.

It accepts the params: method, url, body and response

Example using __jasmine__:

```javascript
import forge from 'mappersmith'
import { install, uninstall, mockRequest } from 'mappersmith/test'

describe('Feature', () => {
  beforeEach(() => install())
  afterEach(() => uninstall())

  it('works', (done) => {
    mockRequest({
      method: 'get',
      url: 'https://my.api.com/users?someParam=true',
      response: {
        body: { allUsers: [{id: 1}] }
      }
    })

    const myManifest = {} // Let's assume I have my manifest here
    const client = forge(myManifest)

    client.User
      .all()
      .then((response) => expect(response.data()).toEqual({ allUsers: [{id: 1}] }))
      .then(done)
  })
})
```

A more complete example:

```javascript
// ...
mockRequest({
  method: 'post',
  url: 'http://example.org/blogs',
  body: 'param1=A&param2=B', // request body
  response: {
    status: 503,
    body: { error: true },
    headers: { 'x-header': 'nope' }
  }
})
// ...
```

## <a name="gateways"></a> Gateways

Mappersmith has a pluggable transport layer and it includes by default two gateways: xhr and http. Mappersmith will pick the correct gateway based on the environment you are running (nodejs or the browser).

You can write your own gateway, take a look at [XHR](https://github.com/tulios/mappersmith/blob/master/src/gateway/xhr.js) for an example. To configure, import the `configs` object and assign the gateway option, like:

```javascript
import { configs } from 'mappersmith'
configs.gateway = MyGateway
```

It's possible to globally configure your gateway through the option `gatewayConfigs`.

### XHR

When running in the browser you can configure `withCredentials` and `configure` to further customize the `XMLHttpRequest` object, example:

```javascript
import { configs } from 'mappersmith'
configs.gatewayConfigs.XHR = {
  withCredentials: true,
  configure(xhr) {
    xhr.ontimeout = () => console.error('timeout!')
  }
}
```

Take a look [here](https://github.com/tulios/mappersmith/blob/master/src/mappersmith.js) for more options.

## <a name="development"></a> Development

### Running unit tests:

```sh
npm run test-browser
npm run test-node
```

### Running integration tests:

```sh
node spec/integration/server.js &
npm run test-browser-integration
npm run test-node-integration
```

### Running all tests

```sh
node spec/integration/server.js &
npm run test
```

## Compile and release

```sh
NODE_ENV=production npm run build
```

## Contributors

Check it out!

https://github.com/tulios/mappersmith/graphs/contributors

## License

See [LICENSE](https://github.com/tulios/mappersmith/blob/master/LICENSE) for more details.
