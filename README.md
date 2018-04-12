# Hydra 3.0

1. [API](#api)
    - [Setup](#setup)
    - [Hydra.create](#api-hydra-create)
    - [hydra.transform](#api-hydra-transform)
      - [Reducers](#reducers)
          - [Accumulator](#accumulator)
          - [PathReducer](#path-reducer)
          - [FunctionReducer](#function-reducer)
          - [EntityReducer](#entity-reducer)
      - [Transforms](#transforms)
          - [Transform](#transform)
          - [TransformMap](#transform-map)
    - [hydra.addEntities](#api-hydra-add-entities)
      - [Entities](#entities)
          - [Entry](#entry)
          - [Source](#source)
              - [StringTemplate](#string-template)
          - [Hash](#hash)
          - [Collection](#collection)
          - [Source](#source)
          - [Schema](#schema)
          - [Entity ComposeReducer](#entity-compose-reducer)
          - [Extending Entities](#extending-entities)
    - [hydra.use](#api-hydra-use)
    - [hydra.addValue](#api-hydra-add-value)

## <a name="api"></a> API

Create a hydra Object:

```js
Hydra.create([options])
```

**Arguments**

| Argument | Type | Description |
|:---|:---|:---|
| `options` | `Object` (_optional_) | This parameter is optional as well as its properties. You may add reducers, values and Entities via its API after creating a hydra instance. |

The following table describes the properties of the `options` object.

| Property | Type | Description |
|:---|:---|:---|
| `reducers` | `Object` | This object includes the reducers you want exposed to hydra Transforms |
| `values` | `Object` | Hash with values you want exposed to every Reducer |
| `entities` | `Object.<string, EntityReducer>` | Models used in hydra |

**RETURNS**

Hydra instance.

<a name="setup-examples"></a>**EXAMPLES**

With no initial config:

```js
const Hydra = require('hydra')
const hydra = Hydra.create()
```

Passing options set:

```js
const Hydra = require('hydra')
const hydra = Hydra.create({
  values: require('./values'),
  entities: require('./entities')
})
```

### <a name="api-hydra-transform"></a> hydra.transform()

**SYNOPSIS**

```js
hydra.transform(Transform, input, options) // promise
hydra.transform(Transform, input, options, done) // nodejs callback function
```

This method will return a **Promise** if the 4th parameter (`done`) is omitted.

**ARGUMENTS**

| Argument | Type | Description |
|:---|:---|:---|
| `Transform` | [Transform](#Transform) | Valid Entity Id |
| `input` | `Object` | value that you wish to transform, if **none**, pass `null` or empty object `{}`. |
| `options` | [TransformOptions](#transform-options) | Options within the scope of the current Transformation |
| `done` | `function` _(optional)_ | error-first callback [Node.js style callback](https://nodejs.org/api/errors.html#errors_node_js_style_callbacks) that has the arguments `(error, result)`, where `result` contains the final resolved [ReducerContext](#reducer-context). The actual transformation result will be inside the `result.value` property |

### <a name="transform"></a> Transform Object

**Description**

A **Transform** Object is a chain of [Reducers](#reducers), where the result of one Reducer gets passed as the input data of the next one. All reducers get executed serially and **asynchronously**.

Transforms can be represented in different ways: 

**String transform:**

```js
const t = '$.'
const t = '$name:string'
const t = '$results[0].users | app.validateUsers | model:User[]'
const t = 'model:User[] | app.validateUsers | $[0]'

// which could be later used as:

hydra.transform(t, someInput).then((acc) => {
  console.log(acc.value) // resolved result
})
```

**Single Function Reducer:**

```js
const t = app.decodeUrl()
```

**Array transform with mixed reducers:**

```js
// just strings
const t = ['$image[0]', 'model:Image']
// string & Function reducers
const t = ['$image[0].url', decodeUrl()]
// piped strings & functions
const t = ['$results[0].users | model:Users', validateUsers()]

// which could be later used as:

hydra.transform(t, someInput).then((acc) => {
  console.log(acc.value) // resolved result
})
```

### <a name="reducers"></a> Reducers

Reducers are used to build a [Transform](#Transform) Object.

Reducers transform a value. They are all executed serially &amp; asynchronously.

Supported reducers:

1. [PathReducer](#path-reducer)
2. [FunctionReducer](#function-reducer)
3. [EntityReducer](#entity-reducer)

#### <a name="path-reducer"></a>  PathReducer
Extracts a path from the current [Accumulator](#accumulator)`.value` . It uses lodash's [_.get](https://lodash.com/docs/4.17.4#get) behind the scenes.


**SYNOPSIS**

```js
$[.|<path>][:<type>]
```

**OPTIONS**

| Option | Type | Description |
|:---|:---|:---|
| `.` | N/A | To reference **root** scope of the current `context`. |
| `path` | [ObjectPath](#object-path) |  Simple Object notation used to extract a path from current `context.value`. |
| `type` | `string` | _optional_ casts the value as given type. Possible values: `*|string|float|int|boolean`. By default (when omitted), it does not cast the value as any type. |

**Examples:**

Traverse an object's structure

```js
const hydra = require('hydra').create()

const input = {
  a: {
    b: 'Hello World'
  }
}

hydra.transform('$a.b', input).then((ctx) => {
  assert.equal(ctx.value, 'Hello World')
})
```

Example at: [examples/reducer-path.js](examples/reducer-path.js)

**Casting:** (Optional)

You may pass a type to which you want to cast your value. Values may be any supported primitive type. If casting is omitted, then the value will be passed as is.

Supported values are: 

`*`, `string`, `float`, `int`, `boolean`

**Examples:**

```js
const input = {
 street: 'River Road',
 active: 'true'
 zipCode: 20010,
 address: {
   state: 'NY',
   country: 'US'
 }
}

hydra.transform('$street:*', input).then((acc) => {
  console.log(acc.value) // street
})

hydra.transform('$active:boolean', input).then((acc) => {
  console.log(acc.value) // true (boolean, not string)
})

hydra.transform('$address:string', input).then((acc) => {
  console.log(acc.value) // [object Object]  
});

hydra.transform('$zipCode:string', input).then((acc) => {
  console.log(acc.value) // '20010' <-- string not number
})
```

### <a name="accumulator">Accumulator</a>

This Object is passed to Reducers and Middleware callbacks; it contains information about the current transformation (or middleware) execution. 

For transformation purposes, you will be accessing the `Accumulator.value` property. This becomes your data source from which you want to apply transformations. The `.value` property **SHOULD** be treated as read-only **-never to be transformed-**, but instead, you use it as your initial source and create a new object from it, this, so your reducers are [pure functions](https://medium.com/javascript-scene/master-the-javascript-interview-what-is-a-pure-function-d1c076bec976#.r4iqvt9f0) which produce no side effects. 

There are many libraries out there that may help you accomplish this in a painless way. 

- [Lodash's fp](https://github.com/lodash/lodash/wiki/FP-Guide)
- [Ramda](http://ramdajs.com/)
- [icepick](https://github.com/aearly/icepick)
- [Mori](http://swannodette.github.io/mori/)
- [Facebook's Immutable](https://facebook.github.io/immutable-js/)
- A search in npm for sure will throw some good results.

**Properties exposed:**

| Key | Type | Description |
|:---|:---|:---|
| `value`  | Object | Value to be transformed |
| `initialValue`  | Object | Original value passed to the entire [Transform](#transform), you can use this value as a reference to the initial value passed to your Transform before any reducer was applied |
| `values`  | Object | Access to the values stored via [hydra.addValue](#api-hydra-add-value) |
| `params`  | Object | Value of the current Entity's params property |
| `locals`  | Object | Value passed from the `locals` _param_ in [hydra.transform](#api-hydra-transform) |
| `reducer`  | Object | Information relative to the current Reducer being executed |

#### <a name="function-reducer"></a>FunctionReducer

A FunctionReducer allows you to execute an asynchronous block of code. Its first parameter is a [Accumulator](#accumulator) Object. The second is an error-first callback ([Node.js style callback](https://nodejs.org/api/errors.html#errors_node_js_style_callbacks)) that has the arguments `(error, value)`, where value will be the _value_ passed to the _next_ transform; that value will eventually be the value of the fully resolved transformation.

**SYNOPSIS**

```js
//es6
const name = (acc:Accumulator, done:function) => {
  done(error:Error, newValue:*)
}
```

**Reducer's arguments**

| Argument | Type | Description |
|:---|:---|:---|
| `acc` | [Accumulator](#accumulator) | Holds information regarding the current reducer's execution. The main property you will be using is `acc.value` which is the current reducer's value. |
| `done` | `Function` _(`error`, `value`)_ | [Node.js style callback](https://nodejs.org/api/errors.html#errors_node_js_style_callbacks), where value is the value to be passed to the next reducer.

**EXAMPLE:**

```js
const reducer = (ctx, next) => {
  next(null, ctx.value + ' World')
}

hydra.transform(reducer, 'Hello').then((ctx) => {
  assert.equal(ctx.value, 'Hello World')
  console.log(ctx.value)
})
```

Example at: [examples/reducer-function.js](examples/reducer-function.js)

#### Higher Order Reducers

Higher Order Reducers are expected to be [Higher-order Functions](https://en.wikipedia.org/wiki/Higher-order_function). In which it **MUST** return a [FunctionReducer](#function-reducer).

```js
const name = (pram1, param2, ...) => (acc:Accumulator, done:function) => {
  done(error:Error, newValue:*)
}
```

**EXAMPLE**

```js
const addStr = (value) => (ctx, next) => {
  next(null, ctx.value + value)
}

hydra.transform(addStr(' World!!'), 'Hello').then((ctx) => {
  assert.equal(ctx.value, 'Hello World!!')
  console.log(ctx.value)
})
```

Example at: [examples/reducer-function-closure.js](examples/reducer-function-closure.js)

#### Array of Reducer Functions

To execute multiple reducers serially, you may pass them wrapped in an Array structure.

```js
const input = {
  a: {
    b: 'Hello World'
  }
}

const toUpperCase = (ctx, next) => {
  next(null, ctx.value.toUpper())
}

hydra.transform(['$a.b', toUpperCase], input).then((ctx) => {
  assert.equal(ctx.value, 'HELLO WORLD')
})
```

Example at: [examples/reducer-array-mixed.js](examples/reducer-array-mixed.js)

The following example extracts the array from the input object, gets its max value, and multiplies by a  given number

```js
const multiplyBy = (factor) => (ctx, next) => {
  next(null, ctx.value * factor)
}

const getMax = () => (ctx, next) => {
  const result = Math.max.apply(null, ctx.value)
  next(null, result)
}

hydra.transform(['$a.b.c', getMax(), multiplyBy(10)], input).then((ctx) => {
  assert.equal(ctx.value, 30)
  console.log(ctx.value)
})
```

Example at: [examples/reducer-array-mixed-2.js](examples/reducer-array-mixed-2.js)

Examples on reducers that were used for the unit tests:
[Unit Test Reducers](https://github.com/Viacom/hydra/blob/master/test/utils/reducers.js)

#### <a name="entity-reducer"></a> EntityReducer

> EntityReducer as the implementation of an Entity

Hydra's real power is displayed by EntityReducers. You may add entities through the `Hydra.create` factory when creating a Hydra instance. After an instance is created, you can also add entities through the [hydra.addEntities](http://README.md#api-hydra-add-entities) method. This method accepts an Object where each key it expects it to be a valid Entity Spec. 

An EntityReducer can be thought of as an implementation of an Entity. The EntityReducer passes the current [ReducerContext](#reducer-context)`.value`  to an Entity. For information on supported (built-in) Entities, go to [Entities Section](#entities).

**SYNOPSIS**

```js
'<EntityType>:<EntityId>[[]]'
```

**OPTIONS**

| Option | Type | Description |
|:---|:---|:---|
| `EntityType` | `String` | Valid Entity type. |
| `EntityID` | `String` | Valid Entity Id. Optionally you may pass a Closed brackets at the end `[]` to indicate mapping |

**Example:**

```js
const input = {
  a: {
    b: 'Hello World'
  }
}

const toUpperCase = (ctx, next) => {
  next(null, ctx.value.toUpper())
}

hydra.addEntities({
  'transform:Foo': ['$a.b', toUpperCase]
})

hydra.transform(['transform:Foo'], input).then((ctx) => {
  assert.equal(ctx.value, 'HELLO WORLD')
})
```

**Collection Mapping:**

Adding `[]` at the end of an entity reducer will map the given entity to each result of the current `context.value`, this assumes `value` is a collection. if the value is not a collection it will ignore the `[]` directive.

**Example:**

```js
const input = {
  a: [
    'Hello World',
    'Hello US',
    'Hello Mexico',
    'Hello Italy',
  ]
}

const toUpperCase = (ctx, next) => {
  next(null, ctx.value.toUpper())
}

hydra.addEntities({
  'transform:toUpperCase': toUpperCase
})

hydra.transform(['$a | transform:toUpperCase[]'], input).then((ctx) => {
  assert.equal(ctx.value[0], 'HELLO WORLD')
  assert.equal(ctx.value[1], 'HELLO US')
  assert.equal(ctx.value[2], 'HELLO MEXICO')
  assert.equal(ctx.value[3], 'HELLO ITALY')
})
```

Reducer entity [Examples](https://github.com/Viacom/hydra/blob/master/test/definitions/integrations.js)


#### <a name="api-hydra-add-entities">hydra.addEntities</a>

In hydra, *entities* are used to model the data. They specify how the data should be transformed. For more information about entities, go to [Entities Section](#entities).

When adding entity objects to hydra, only valid (registered) entity types are allowed.

**SYNOPSIS**

```js
hydra.addEntities({
  '<EntityType>:<EntityId>': { ... },
  '<EntityType>:<EntityId>': { ... },
  ...
})
```

**OPTIONS**

| Part | Type | Description |
|:---|:---|:---|
| `EntityType` | `string` | valid entity type to associate the EntityObject |
| `EntityId` | `string` | unique entity id associated with the EntityObject |

**Best Practices: (Recommendations)**

```js
const badReducer = () => (acc, done) => {
  // this is BAD - dont be this person! 
  // never ever modify the value object.
  acc.value[1].username = 'foo'

  // keep in mind JS is by reference
  // so this means this is also BAD
  const image = acc.value[1]
  image.username = 'foo'

  // pass value to next reducer
  done(null, acc.value)
}

// this is better
const fp = require('lodash/fp')
const goodReducer = () => (acc, done) => {
  // https://github.com/lodash/lodash/wiki/FP-Guide
  // this will cause no side effects
  const value = fp.set('[1].username', 'foo', acc.value)

  // pass value to next reducer
  done(null, value)
}

```

### <a name="entities">Entities</a>

Entities are artifacts we can use to transform our data. An entity is represented by a data structure (spec) that defines how the entity behaves. 

Entities may be added in two ways: 

1. On the hydra constructor as explained in [setup examples](#setup-examples).
2. Through the `hydra.addEntities()` method explained in [addEntities](#api-hydra-add-entities).

#### <a name="built-in-entities">Built-in Entities</a>

Hydra comes with the following built-in entities: 

- [Entry](#entry)
- [Source](#source)
- [Hash](#hash)
- [Collection](#collection)
- [Control](#control)
- [Schema](#schema)

All entities have a common API (except for [Transform](#entity-transform)), each entity builds on top of this common API:

```js
{
  // executes --before-- everything else
  before: Transform,
  
  // executes --after-- everything else
  after: Transform,
  
  // executes in case there is an error at any
  // point of the entire transformation
  error: Transform,
  
  // this object allows you to store and eventually
  // access it at any given time on any reducer
  params: Object
}
```

**Properties exposed:**

| Key | Type | Description |
|:---|:---|:---|
| `before`  | [Transform](#transform) | Transform to be resolved **before** the execution of `context` |
| `after`   | [Transform](#transform) | Transform to be resolved **after** the execution of `context` |
| `error`   | [Transform](#transform) | Transform to be resolved in case of an error |
| `params`    | `Object` | User defined Hash that will be passed to every transform within the context of the Transform's execution |

#### <a name="entry"></a> Entry

An Entry entity is where your data manipulation starts. Ideally, entries are calling more complex entities and should be simple on their own.

**SYNOPSIS**

```js
{
  before: Transform,

  value: Transform,

  after: Transform,
  error: Transform,
  params: Object
}
```

**Properties exposed:**

| Key | Type | Description |
|:---|:---|:---|
| `before`  | [Transform](#transform) | Transform to be resolved **before** the execution of `context` |
| `context` | [Transform](#transform) | The value this Object resolves to |
| `after`   | [Transform](#transform) | Transform to be resolved **after** the execution of `context` |
| `error`   | [Transform](#transform) | Transform to be resolved in case of an error |
| `params`    | `Object` | User defined Hash that will be passed to every transform within the context of the Transform's execution |

##### Entry.value

**EXAMPLE:**

Using the `value` property to transform an input

```js
const input = {
  a: {
    b: {
      c: [1, 2, 3]
    }
  }
}

const getMax = (ctx, next) => {
  const result = Math.max.apply(null, ctx.value)
  next(null, result)
}

const multiplyBy = (number) => (ctx, next) => {
  const newValue = ctx.value * number
  next(null, newValue)
}

hydra.addEntities({
  'entry:foo': {
    value: ['$a.b.c', getMax, multiplyBy(10)]
  }
})

hydra.transform('entry:foo', input).then((ctx) => {
  assert.equal(ctx.value, 30)
  console.log('result:', ctx.value)
})
```

Example at: [examples/entity-entry-basic.js](examples/entity-entry-basic.js)

##### Entry.before

Check if value passed to entity is Array

```js
const isArray = () => (ctx, next) => {
  if (ctx.value instanceof Array) {
    // if the value is valid, then just pass it along
    return next(null, ctx.value)
  }

  // notice how we pass this Error object as the FIRST parameter,
  // this tells Hydra there was an error, and to treat it as such.
  next(new Error(`${ctx.value} should be an Array`))
}

hydra.addEntities({
  'entry:foo': {
    before: isArray(),
    value: '$.'
  }
})

hydra.transform('entry:foo', [3, 15, 6, 3, 8]).then((ctx) => {
  console.log(ctx.value)
  // [ 3, 15, 6, 3, 8 ]
})
```

Example at: [examples/entity-entry-before.js](examples/entity-entry-before.js)

##### Entry.after

We can move this same logic on the `after` Transform.

```js
const isArray = () => (ctx, next) => {
  if (ctx.value instanceof Array) {
    // if the value is valid, then just pass it along
    return next(null, ctx.value)
  }

  // notice how we pass this Error object as the FIRST parameter,
  // this tells Hydra there was an error, and to treat it as such.
  next(new Error(`${ctx.value} should be an Array`))
}

hydra.addEntities({
  'entry:foo': {
    value: '$a.b',
    after: isArray()
  }
})

const input = {
  a: {
    b: [3, 15, 6, 3, 8]
  }
}

hydra.transform('entry:foo', input).then((ctx) => {
  console.log(ctx.value)
  // [ 3, 15, 6, 3, 8 ]
})
```

Example at: [examples/entity-entry-after.js](examples/entity-entry-after.js)

##### Entry.error

Any error that happens within the scope of the Entity can be handled by the `error` Transform. To respect the API, error reducers have the same api, and the value of the error is under `ctx.value`.

**Error handling**

Passing a a value as the second argument will stop the propagation of the error.

Let's resolve to a NON array value and see how this would be handled. 

```js
const isArray = () => (ctx, next) => {
  if (ctx.value instanceof Array) {
    // if the value is valid, then just pass it along
    return next(null, ctx.value)
  }

  // notice how we pass this Error object as the FIRST parameter,
  // this tells Hydra there was an error, and to treat it as such.
  next(new Error(`${ctx.value} should be an Array`))
}

hydra.addEntities({
  'entry:foo': {
    // points to a NON Array value
    value: '$a',
    after: isArray(),
    error: (ctx, next) => {
      console.log('Value is invalid, resolving to empty array')
      // passing a a value as the second argument
      // will stop the propagation of the error
      next(null, [])
    }
  }
})

const input = {
  a: {
    b: [3, 15, 6, 3, 8]
  }
}

hydra.transform('entry:foo', input).then((ctx) => {
  console.log(ctx.value)
  // Value is invalid, resolving to empty array
  // []
})
```

Example at: [examples/entity-entry-error-handled.js](examples/entity-entry-error-handled.js)

**Re-throw the array to be handled somewhere else**

```js
const logError = () => (errCtx, next) => {
  // errCtx.value holds the actual value
  console.log(errCtx.value.toString())

  // if we wish to bubble it up, then pass it to
  // the next() as the first parameter
  next(errCtx.value)

  // if we wished not to bubble it, we could pass
  // an empty first param, and a second value to
  // be used as the final resolved value
  // next(null, false) <-- this is just an example
}

hydra.addEntities({
  'entry:foo': {
    value: '$a',
    after: isArray(),
    error: logError()
  }
})

const input = {
  a: {
    b: [3, 15, 6, 3, 8]
  }
}

hydra.transform('entry:foo', input)
  .catch((error) => {
    console.log('got ya!')
    // Error: [object Object] should be an Array
    // got ya!
  })
```

Example at: [examples/entity-entry-error-rethrow.js](examples/entity-entry-error-rethrow.js)

Let's resolve to some value, in order to prevent the transform from failing.

```js
const resolveTo = (value) => (errCtx, next) => {
  // since we don't pass the error back
  // it will resolve to the new value
  next(null, value)
}

hydra.addEntities({
  'entry:foo': {
    value: '$a',
    after: isArray(),
    // resolving the value to empty array
    error: resolveTo([])
  }
})

const input = {
  a: {
    b: [3, 15, 6, 3, 8]
  }
}

hydra.transform('entry:foo', input)
  .then((ctx) => {
    assert.deepEqual(ctx.value, [])
    console.log(ctx.value)
    // []
  })
```

Example at: [examples/entity-entry-error-resolved.js](examples/entity-entry-error-resolved.js)

For examples of entry entities please look at the ones used under the [Examples](https://github.com/Viacom/hydra/blob/master/examples), on the unit tests: [Source Definitions](https://github.com/Viacom/hydra/blob/master/test/definitions/entry.js) and [Integration Examples](https://github.com/Viacom/hydra/blob/master/test/definitions/integrations.js)

#### <a name="source"></a> Source

Requests a remote source; uses [Request](https://github.com/request/request) behind the scenes. The features supported by Request are also exposed/supported by Source.

**SYNOPSIS**

```js
{
  before: Transform,

  url: StringTemplate,
  options: Object,
  beforeRequest: Transform,
  
  after: Transform,
  error: Transform,
  params: Object
}
```

**Properties exposed:**

| Key | Type | Description |
|:---|:---|:---|
| `before` | [Transform](#transform) | Transform to be resolved in **before** making the request |
| `url`   | [StringTemplate](#string-template) | String value to resolve the request's url |
| `options` | `Object` | Request's options. This maps directly to [request.js](https://github.com/request/request) options
| `beforeRequest` | [Transform](#transform) | `acc.value` at this point will be the request options object being passed to the final request, you may do any modifications here, and pass to the next reducer |
| `after` | [Transform](#transform) | Transform to be resolved in **after** making the request |
| `error`   | [Transform](#transform) | Transform to be resolved in case of an error |
| `params`    | `Object` | User defined Hash that will be passed to every transform within the context of the Transform's execution |

##### Source.url

**GitHub API - list organization info**

(info on this API: [https://api.github.com/orgs/nodejs](https://api.github.com/orgs/nodejs))

Fetch an organization's information.

```js
hydra.addEntities({
  'source:getOrgInfo': {
    url: 'https://api.github.com/orgs/nodejs',
    options: {
      headers: {
        'User-Agent': 'hydra'
      }
    }
  }
})
```

hydra.transform('source:getOrgInfo', {}).then((ctx) => {
  console.log(ctx.value) // entire result from https://api.github.com/orgs/nodejs
})

Example at: [examples/entity-source-basic.js](examples/entity-source-basic.js)

##### <a name="string-template"></a> StringTemplate

StringTemplate is a string that supports a **minimal** templating system. You may inject any value into the string by enclosing it within `{ObjectPath}` curly braces. **The context of the string is the Source's Accumulator Object**, meaning you have access to any property within it. 

Using `acc.value` property to make the url dynamic.

```js
hydra.addEntities({
  // dynamic search, which uses StringTemplate's simple 
  // templating system {value} to inject the search 
  // value into the URL string
  'source:getNodeOrgInfo': {
    url: 'https://api.github.com/orgs/{value.organization}',
    // this object will be passed to request.js
    options: {
      headers: {
        'User-Agent': 'request'
      }
    }
  }
})

// second parameter to transform is the initial value
transform('entry:getNodeOrgInfo', {
  organization: 'nodejs'
}).then((acc) => {
  // outputs full result from the remote source
  console.log(acc.value) 
})
```

Using `acc.locals` property to make the url dynamic.

```js
hydra.addEntities({
  // dynamic search, which uses StringTemplate's simple 
  // templating system {value} to inject the search 
  // value into the URL string
  'source:getNodeOrgInfo': {
    url: 'https://api.github.com/orgs/{locals.organization}',
    // this object will be passed to request.js
    options: {
      headers: {
        'User-Agent': 'request'
      }
    }
  }
})

// third parameter of transform is the options object 
// where we can pass the `locals` value
transform('entry:getNodeOrgInfo', null, {
  locals: {
    organization: 'nodejs'
  }
}).then((acc) => {
  // outputs full result from the remote source
  console.log(acc.value) 
})
```

Example at: [examples/entity-source-string-template.js](examples/entity-source-string-template.js)

##### Source.beforeRequest

There are times where you may want to process the `source.options` object before being actually passed to send the request. 

In this example, all we are doing is providing the headers object through a reducer, but a more real use case could be to set up [OAuth Signing](https://www.npmjs.com/package/request#oauth-signing).

```js
hydra.addEntities({
  'source:getOrgInfo': {
    url: 'https://api.github.com/orgs/{value}',
    beforeRequest: (ctx, next) => {
      // ctx.value holds reference to source.options
      const options = _.assign({}, ctx.value, {
        headers: {
          'User-Agent': 'hydra'
        }
      })

      next(null, options)
    }
  }
})

hydra.transform('source:getOrgInfo', 'nodejs').then((ctx) => {
  console.log(ctx.value)
  // entire result from https://api.github.com/orgs/nodejs
})
```

Example at: [examples/entity-source-before-request.js](examples/entity-source-before-request.js)

For more examples of source entities please look at the ones used under the [Examples](https://github.com/Viacom/hydra/blob/master/examples), on the unit tests: [Source Definitions](https://github.com/Viacom/hydra/blob/master/test/definitions/sources.js), and [Integration Examples](https://github.com/Viacom/hydra/blob/master/test/definitions/integrations.js)

#### <a name="hash"></a> Hash

A Hash Entity transforms a _Hash_ like data structure. It enables you to manipulate the keys a Hash has. 

To prevent unexpected results, **Hash** only can process **Plain Objects**, meaning an object created by the Object constructor. 

Hash Entities expose a set of reducers you may apply to them: mapKeys, omitKeys, pickKeys, addKeys, addValues. You can apply one or more of the available reducers to your Hash. Keep in mind that those reducers will always be executed in a specific order:

```js
mapKeys -> omitKeys -> pickKeys -> addKeys -> addValues
```

If you wish to have more control over the execution you may use the [compose](#entity-compose-reducer) reducer.

**NOTE**: It is meant to operate only on Hash like objects, if its context resolves to a non Hash type it will **throw an error**.

**SYNOPSIS**

```js
{
  before: Transform,

  value: Transform,
  mapKeys: TransformMap,
  omitKeys: String[],
  pickKeys: String[],
  addKeys: TransformMap,
  addValues: Object,
  compose: ComposeModifier[],
  
  after: Transform,
  error: Transform,
  params: Object,
}
```

**Properties exposed:**

| Key | Type | Description |
|:---|:---|:---|
| `before`  | [Transform](#transform) | Transform to be resolved **before** the execution of `context` |
| `context` | [Transform](#transform) | The value this Object resolves to |
| `mapKeys` | [TransformMap](#transform-map) | Map to a new set of key/values, each value accepts a Transform |
| `omitKeys` | `String`[] | Omits keys from context.value (Array of strings) |
| `pickKeys` | `String`[] | Picks keys from context.value (Array of strings) |
| `addKeys` | [TransformMap](#transform-map) | Add/Override key/values, each value accepts a Transform |
| `addValues`| `Object` | Add/Override hard-coded key/values |
| `compose`| `Array<` [ComposeReducer](#compose-reducer) `>` | Modify the value of `context` through a custom order of `ComposeReducer` Objects, think of it as a [Compose/Flow Operation](https://en.wikipedia.org/wiki/Function_composition_(computer_science)) where the result of one gets passed to the next one|
| `after`   | [Transform](#transform) | Transform to be resolved **after** the execution of `context` |
| `error`   | [Transform](#transform) | Transform to be resolved in case of an error |
| `params`    | `Object` | User defined Hash that will be passed to every transform within the context of the Transform's execution |

### Hash.value

Resolves a context of a hash

```js
const hydra = require('../').create()
const assert = require('assert')

const input = {
  a: {
    b: {
      c: 'Hello',
      d: ' World!!'
    }
  }
}

hydra.addEntities({
  'hash:helloWorld': {
    value: '$a.b'
  }
})

hydra.transform('hash:helloWorld', input).then((ctx) => {
  assert.deepEqual(ctx.value, {
    c: 'Hello',
    d: ' World!!'
  })
  console.log('result:', ctx.value)
  // result: { c: 'Hello', d: ' World!!' }
})
```

Example at: [examples/entity-hash-context.js](examples/entity-hash-context.js)

### Hash.mapKeys

Map to a new set of key/value pairs. Each value accepts a Transform.

Going back to our GitHub API examples, let's map some keys from the result of a source:

```js
hydra.addEntities({
  'source:getOrgInfo': {
    url: 'https://api.github.com/orgs/nodejs',
    options: { headers: { 'User-Agent': 'hydra' } }
  },
  'hash:OrgInfo': {
    mapKeys: {
      reposUrl: '$repos_url',
      eventsUrl: '$events_url',
      avatarUrl: '$avatar_url',
      orgName: '$name',
      blogUrl: '$blog'
    }
  }
})

hydra.transform('source:getOrgInfo | hash:OrgInfo').then((ctx) => {
  console.log(ctx.value)
  // {
  //  reposUrl: 'https://api.github.com/orgs/nodejs/repos',
  //  eventsUrl: 'https://api.github.com/orgs/nodejs/events',
  //  avatarUrl: 'https://avatars0.githubusercontent.com/u/9950313?v=3',
  //  orgName: 'Node.js Foundation',
  //  blogUrl: 'https://nodejs.org/foundation/'
  // }
})
```

Example at: [examples/entity-hash-mapKeys.js](examples/entity-hash-mapKeys.js)

### Hash.addKeys

Adds Keys to the current Hash value. If an added key already exists, it will be overridden.

Hash.addKeys is very similar to Hash.mapKeys, but what is different about it is that `mapKeys` will ONLY map the keys you give it, whereas `addKeys` will ADD/APPEND new keys to your existing `context.value`. You may think of it as an _extend_ operation. 

```js
hydra.addEntities({
  'entry:orgInfo': {
    value: 'source:getOrgInfo | hash:OrgInfo | hash:OrgInfoCustom'
  },
  'source:getOrgInfo': {
    url: 'https://api.github.com/orgs/nodejs',
    options: { headers: { 'User-Agent': 'hydra' } }
  },
  'hash:OrgInfo': {
    mapKeys: {
      reposUrl: '$repos_url',
      eventsUrl: '$events_url',
      avatarUrl: '$avatar_url',
      orgName: '$name',
      blogUrl: '$blog'
    }
  },
  'hash:OrgInfoCustom': {
    addKeys: {
      // notice we are extracting from the new key
      // made on the hash:OrgInfo entity
      avatarUrlDuplicate: '$avatarUrl'
    }
  }
})

const expectedResult = {
  reposUrl: 'https://api.github.com/orgs/nodejs/repos',
  eventsUrl: 'https://api.github.com/orgs/nodejs/events',
  avatarUrl: 'https://avatars0.githubusercontent.com/u/9950313?v=3',
  orgName: 'Node.js Foundation',
  blogUrl: 'https://nodejs.org/foundation/',
  // this key was added through hash:OrgInfoCustom
  avatarUrlDuplicate: 'https://avatars0.githubusercontent.com/u/9950313?v=3'
}

hydra.transform('entry:orgInfo', { org: 'nodejs' }).then((ctx) => {
  assert.deepEqual(ctx.value, expectedResult)
  console.log(ctx.value)
  /*
  {
    reposUrl: 'https://api.github.com/orgs/nodejs/repos',
    eventsUrl: 'https://api.github.com/orgs/nodejs/events',
    avatarUrl: 'https://avatars0.githubusercontent.com/u/9950313?v=3',
    orgName: 'Node.js Foundation',
    blogUrl: 'https://nodejs.org/foundation/',
    avatarUrlDuplicate: 'https://avatars0.githubusercontent.com/u/9950313?v=3'
  }
  */
})
```

Example at: [examples/entity-hash-addKeys.js](examples/entity-hash-addKeys.js)

### Hash.pickKeys

Picks a list of keys from the current Hash Value.

The next example is similar to our previous example, but instead of mapping key/value pairs, let's just pick some of the keys:

```js
hydra.addEntities({
  'entry:orgInfo': {
    value: 'source:getOrgInfo | hash:OrgInfo'
  },
  'source:getOrgInfo': {
    url: 'https://api.github.com/orgs/{value.org}',
    options: { headers: { 'User-Agent': 'hydra' } }
  },
  'hash:OrgInfo': {
    // notice this is an array not a Transform or Object
    pickKeys: [
      'name',
      'blog'
    ]
  }
})

// keys came out intact
const expectedResult = {
  name: 'Node.js Foundation',
  blog: 'https://nodejs.org/foundation/'
}

hydra.transform('entry:orgInfo', { org: 'nodejs' }).then((ctx) => {
  console.log(ctx.value)
  assert.deepEqual(ctx.value, expectedResult)
  /*
  {
    name: 'Node.js Foundation',
    blog: 'https://nodejs.org/foundation/'
  }
  */
})
```

Example at: [examples/entity-hash-pickKeys.js](examples/entity-hash-pickKeys.js)

### Hash.omitKeys

Omits keys from the Hash value.

In case we want to only **omit** some keys, and let the rest pass through:

```js
hydra.addEntities({
  'entry:orgInfo': {
    value: 'source:getOrgInfo | hash:OrgInfo'
  },
  'source:getOrgInfo': {
    url: 'https://api.github.com/orgs/{value.org}',
    options: { headers: { 'User-Agent': 'hydra' } }
  },
  'hash:OrgInfo': {
    // notice this is an array, not a Transform or Object
    omitKeys: [
      'repos_url',
      'events_url',
      'hooks_url',
      'issues_url',
      'members_url',
      'public_members_url',
      'avatar_url',
      'description',
      'name',
      'company',
      'blog',
      'location',
      'email',
      'has_organization_projects',
      'has_repository_projects',
      'public_repos',
      'public_gists',
      'followers',
      'following',
      'html_url',
      'created_at',
      'updated_at',
      'type'
    ]
  }
})

// keys came out intact
const expectedResult = {
  login: 'nodejs',
  id: 9950313,
  url: 'https://api.github.com/orgs/nodejs'
}

hydra.transform('entry:orgInfo', { org: 'nodejs' }).then((ctx) => {
  console.log(ctx.value)
  assert.deepEqual(ctx.value, expectedResult)
  /*
  {
    login: 'nodejs',
    id: 9950313,
    url: 'https://api.github.com/orgs/nodejs'
  }
  */
})
```

Example at: [examples/entity-hash-omitKeys.js](examples/entity-hash-omitKeys.js)

### Hash.addValues

Adds hard-coded values to the Hash value.

Sometimes you just want to add a hard-coded value to your current `context.value`

```js
hydra.addEntities({
  'hash:addValues': {
    addValues: {
      foo: 'value',
      bar: true, 
      obj: { 
        a: 'a'
      }
    } 
  }
})

// keys came out intact
const expectedResult = {
  foo: 'value',
  bar: true, 
  obj: { 
    a: 'a'
  }
}


hydra.transform('hash:addValues').then((ctx) => {
  expect(ctx.value).toEqual(expectedResult)
})
```

### Hash - adding multiple reducers

You can add multiple reducers to your Hash Spec.

```js
const toUpperCase = (ctx, next) => {
  const result = ctx.value.toUpperCase()
  next(null, result)
}

hydra.addEntities({
  'entry:orgInfo': {
    value: 'source:getOrgInfo | hash:OrgInfo'
  },
  'source:getOrgInfo': {
    url: 'https://api.github.com/orgs/{value.org}',
    options: { headers: { 'User-Agent': 'hydra' } }
  },
  'hash:OrgInfo': {
    mapKeys: {
      reposUrl: '$repos_url',
      eventsUrl: '$events_url',
      avatarUrl: '$avatar_url',
      orgName: '$name',
      blogUrl: '$blog',
    },
    addKeys: {
      upperCaseName: [`$name`, toUpperCase]
    },
    omitKeys: ['eventsUrl'],
    pickKeys: ['reposUrl', 'upperCaseName'],
    addValues: {
      info: 'This is a test'
    }
  }
})

const expectedResult = {
  eventsUrl: 'https://api.github.com/orgs/nodejs/events',
  orgName: 'NODE.JS FOUNDATION',
  info: 'This is a test'
}

hydra.transform('entry:orgInfo', { org: 'nodejs' }).then((ctx) => {
  expect(ctx.value).toEqual(expectedResult)
})
```

For examples of hash entities please look at the ones used under the [Examples](https://github.com/Viacom/hydra/blob/master/examples), on the unit tests: [Source Definitions](https://github.com/Viacom/hydra/blob/master/test/definitions/hash.js), and [Integration Examples](https://github.com/Viacom/hydra/blob/master/test/definitions/integrations.js)

#### <a name="transform-map"></a> TransformMap

This structure allows you to map key/value pairs. Where each value is of type [Transform](#transform).

**SYNOPSIS**

```js
{
  key1: Transform,
  key2: Transform,
  ...
}
```

A TransformMap is just a **hash** where each key is of _type_ [Transform](#transform).

#### <a name="collection"></a> Collection

A Collection Entity enables you to operate over an Array. Its API provides with basic reducers to manipulate the elements in the array.

Collection Entities expose a set of reducers you may apply to them: map, find, filter. They are all executed in a [specific order](#collection-reducers-order). If you wish to have more control over the execution you may use the [compose](#compose-reducer) reducer.

**IMPORTANT:**
Keep in mind that in Hydra, **all** operations are Asynchronous. If your operations do NOT need to be asynchronous, iterating over a large Array might result in slower execution. In such cases, consider using a reducer function that where you can implement a synchronous solution.

**SYNOPSIS**

```js
{
  before: Transform,

  value: Transform,
  filter: Transform,
  map: Transform,
  find: Transform,
  compose: ComposeModifier[],
  
  after: Transform,
  error: Transform,
  params: Object,
}
```

**Properties exposed:**

| Key | Type | Description |
|:---|:---|:---|
| `before`  | [Transform](#transform) | Transform to be resolved **before** the execution of `context` |
| `context` | [Transform](#transform) | The value this Object resolves to |
| `map` | [Transform](#transform) | Maps the items of an Array, **NOTE**: this operation happens asynchronously, so be careful to use it only when async operation is needed, otherwise use a synchronous equivalent (native array filter or third party solution) |
| `find` | [Transform](#transform) | Find an item in the Array, **NOTE**: this operation happens asynchronously, so be careful to use it only when async operation is needed, otherwise use a synchronous equivalent (native array filter or third party solution) |
| `filter` | [Transform](#transform) | Filters the items of an Array, **NOTE**: this operation happens asynchronously, so be careful to use it only when async operation is needed, otherwise use a synchronous equivalent (native array filter or third party solution) |
| `compose`| `Array<` [ComposeReducer](#compose-reducer) `>` | Modify the value of `context` through a custom order of `ComposeReducer` Objects, think of it as a [Compose/Flow Operation](https://en.wikipedia.org/wiki/Function_composition_(computer_science)) where the result of one gets passed to the next one|
| `after`   | [Transform](#transform) | Transform to be resolved **after** the execution of `context` |
| `error`   | [Transform](#transform) | Transform to be resolved in case of an error |
| `params`    | `Object` | User defined Hash that will be passed to every transform within the context of the Transform's execution |

<a name="collection-reducers-order"></a>The order of execution of is: 

```js
filter -> map -> find
```

##### Collection.map

Maps a transformation to each element in a collection

Let's fetch all the repositories that the [NodeJs](https://github.com/nodejs) Org has available at the API: [https://api.github.com/orgs/nodejs/repos](https://api.github.com/orgs/nodejs/repos).

Now that we have the result of the fetch, let's now map each item and extract only one of its properties.

```js
hydra.addEntities({
  'source:getOrgRepositories': {
    url: 'https://api.github.com/orgs/nodejs/repos'
  },
  'collection:getRepositoryTagsUrl': {
    map: '$tags_url'
  }
})

hydra.transform('source:getGist | collection:getRepositoryTagsUrl', {}).then((ctx) => {
  console.log(ctx.value)
  /*
  [
    https://api.github.com/repos/nodejs/http-parser/tags,
    https://api.github.com/repos/nodejs/node-v0.x-archive/tags,
    https://api.github.com/repos/nodejs/node-gyp/tags,
    https://api.github.com/repos/nodejs/readable-stream/tags,
    https://api.github.com/repos/nodejs/node-addon-examples/tags,
    https://api.github.com/repos/nodejs/nan/tags,
    ...
  ]
  */
})
```

The above example is fairly simple. The following example hits each of these urls, and gets information from them. 

**Get latest tag of each repository:**

_for the purpose of this example, let's imagine that GitHub does not provide the url api to get the list of tags._

```js
hydra.addEntities({
  'source:getOrgRepositories': {
    url: 'https://api.github.com/orgs/nodejs/repos'
  },
  'source:getLatestTag': {
    // here we are injecting the current context.value 
    // that was passed to the source
    url: 'https://api.github.com/repos/nodejs/{value}/tags'
  },
  'collection:getRepositoryLatestTag': {
    // magic!! here we are telling it to map each 
    // repository.name to a source:getLatestTag, and return the entire source
    map: '$name | source:getLatestTag'
  }
})

hydra.transform('source:getOrgRepositories | collection:getRepositoryLatestTag', {}).then((ctx) => {
  console.log(ctx.value)
  /*
  [
    [  // repo
      {
        "name": "v2.7.1",
        "zipball_url": "https://api.github.com/repos/nodejs/http-parser/zipball/v2.7.1",
        "tarball_url": "https://api.github.com/repos/nodejs/http-parser/tarball/v2.7.1",
        "commit": {...}
        ...
      },
      ...
    ],
    [ // repo
      {
      "name": "works",
      "zipball_url": "https://api.github.com/repos/nodejs/node-v0.x-archive/zipball/works",
      "tarball_url": "https://api.github.com/repos/nodejs/node-v0.x-archive/tarball/works",
      "commit": {...}
      ...
      },
      ...
    ],
    ... // more repos
  ]
  */
})
```

Well, that's perfect! What if we only wanted the latest tag github has on each repository?

```js
hydra.addEntities({
  'source:getOrgRepositories': {
    url: 'https://api.github.com/orgs/nodejs/repos'
  },
  'source:getLatestTag': {
    // here we are injecting the current context.value 
    // that was passed to the source
    url: 'https://api.github.com/repos/nodejs/{value}/tags',
    options
  },
  'collection:getRepositoryLatestTag': {
    // notice similar to previous example, BUT
    // we add a third reducer at the end to get 
    // the first element of each tag result,
    // and the name of it
    map: '$name | source:getLatestTag | $[0].name'
  }
})

hydra.transform('source:getOrgRepositories | collection:getRepositoryLatestTag', {}).then((ctx) => {
  console.log(ctx.value)
  /*
  [
    "v2.7.1",
    "works",
    "v3.6.0",
    "v2.2.9",
    null,
    "v2.6.2",
    ...
  ]
  */
})
```

##### Collection.filter

Creates a new array with all elements that pass the test implemented by the provided Transform

Let's filter some data. Let's get all the repos that have more than 100 stargazers.

```js
hydra.addEntities({
  'source:getOrgRepositories': {
    url: 'https://api.github.com/orgs/nodejs/repos'
  },
  'collection:getRepositoryUrl': {
    filter: (ctx, next) => {
      next(null, ctx.value > 100)
    }
  }
})

hydra.transform('source:getGist', {}).then((ctx) => {
  console.log(ctx.value)
  /*
  [
    https://api.github.com/repos/nodejs/http-parser,
    https://api.github.com/repos/nodejs/node-v0.x-archive,
    https://api.github.com/repos/nodejs/node-gyp,
    https://api.github.com/repos/nodejs/readable-stream,
    https://api.github.com/repos/nodejs/node-addon-examples,
    https://api.github.com/repos/nodejs/nan,
    ...
  ]
  */
})
```

Because `filter` accepts a Transform, you could use it to check if a property evaluates to a [Truthy](https://developer.mozilla.org/en-US/docs/Glossary/Truthy) value. 

Get all the repos that are actually forks. In this case, because the `fork` property is a boolean, then we can do the following:

```js
hydra.addEntities({
  'source:getOrgRepositories': {
    url: 'https://api.github.com/orgs/nodejs/repos'
  },
  'collection:getRepositoryUrl': {
    filter: '$fork'
  }
})

hydra.transform('source:getGist', {}).then((ctx) => {
  console.log(ctx.value)
  /*
  [
    {
      "id": 28619960,
      "name": "build-container-sync",
      "full_name": "nodejs/build-container-sync",
      ...
      "fork": true
    }, 
    {
      "id": 30464379,
      "name": "nodejs-es",
      "full_name": "nodejs/nodejs-es",
      ...
      "fork": true
    }
  ]
  */
})
```

##### Collection.find

Returns the value of the first element in the array that satisfies the provided testing Transform. Otherwise, `undefined` is returned.

**Find the a repository with name equals: node**

```js
hydra.addEntities({
  'source:repos': {
    url: 'https://api.github.com/orgs/nodejs/repos',
    options: {
      headers: {
        'User-Agent': 'request'
      }
    }
  },
  'collection:getNodeRepo': {
    before: 'source:repos',
    find: (ctx, next) => {
      // notice we are checking against the property -name- 
      next(null, ctx.value.name === 'node')
    }
  }
})

hydra.transform('source:repos | collection:getNodeRepo', {}).then((ctx) => {
  console.log(ctx.value)
  /*
  {
    "id": 27193779,
    "name": "node",
    "full_name": "nodejs/node",
    "owner": { ... },
    "private": false,
    "html_url": https://github.com/nodejs/node,
    "description": "Node.js JavaScript runtime :sparkles::turtle::rocket::sparkles:",
    ...
  }
  */
})
```

##### Collection.compose

`Collection.compose` receives an array of **modifiers**  (filter, map, find). You may add as many modifiers as you need, in any order, by _composition_. 


**IMPORTANT:** The order of your modifiers is important. Keep in mind that `Collection.find` returns the **matched** element. However, the map and filter modifiers expect an array. If your matched item is **NOT** an Array, the Entity will **throw an error**. 

Let's refactor the previous Collection.find example:

```js
const isEqualTo = (match) => (ctx, next) => {
  next(null, ctx.value === match)
}

hydra.addEntities({
  'source:repos': {
    url: 'https://api.github.com/orgs/nodejs/repos',
    options: {
      headers: {
        'User-Agent': 'request'
      }
    }
  },
  'collection:getNodeRepo': {
    before: 'source:repos',
    compose: [{
      // passing the value of property -name- to the
      // next reducer, which will compare to a given value 
      find: ['$name', isEqualTo('node')]
    }]
  }
};

hydra.transform('source:repos | collection:getNodeRepo', {}).then((ctx) => {
  console.log(ctx.value)
  /*
  {
    "id": 27193779,
    "name": "node",
    "full_name": "nodejs/node",
    "owner": { ... },
    "private": false,
    "html_url": https://github.com/nodejs/node,
    "description": "Node.js JavaScript runtime :sparkles::turtle::rocket::sparkles:",
    ...
  }
  */
})
```

**Get all forks and map them to a Hash entity**

```js
const isEqualTo = (match) => (ctx, next) => {
  next(null, ctx.value === match)
}

hydra.addEntities({
  'source:repos': {
    url: 'https://api.github.com/orgs/nodejs/repos',
    options: {
      headers: {
        'User-Agent': 'request'
      }
    }
  },
  'hash:repositorySummary': {
    pickKeys: ['id', 'name', 'homepage', 'description']
  },
  'collection:forkedReposSummary': {
    compose: [
      { filter: '$fork'},
      { map: 'hash:repositorySummary' }
    ]
  }
};

hydra.transform('source:repos | collection:forkedReposSummary', {}).then((ctx) => {
  console.log(ctx.value)
  /*
  [
    {
      "id": 28619960,
      "name": "build-container-sync",
      "homepage": null,
      "description": null
    },
    {
      "id": 30464379,
      "name": "nodejs-es",
      "homepage": "",
      "description": "Localización y traducción de io.js a Español"
    }
  ]
  */
})
```

For more examples of collection entities please look at the ones used under the [Examples](https://github.com/Viacom/hydra/blob/master/examples), on the unit tests: [Source Definitions](https://github.com/Viacom/hydra/blob/master/test/definitions/collection.js), and [Integration Examples](https://github.com/Viacom/hydra/blob/master/test/definitions/integrations.js)

#### <a name="control"></a> Control

Flow Control Entity, allows you have control over the flow of your transformations.

**SYNOPSIS**

```js
{
  select: [
    { case: Transform, do: Transform },
    ...
    { default: Transform }
  ],
  before: Transform,
  after: Transform,
  error: Transform,
  params: Object,
}
```

**Properties exposed:**

| Key | Type | Description |
|:---|:---|:---|
| `select` | `Array`<Case Statements> | Array of case statements, and a default fallback |
| `options` | `Object` | Avj's [options](https://github.com/epoberezkin/ajv#options) object
| `params`    | `Object` | User defined Hash that will be passed to every transform within the context of the Transform's execution |
| `before` | [Transform](#transform) | Transform to be resolved in **before** making the request |
| `after` | [Transform](#transform) | Transform to be resolved in **after** making the request |
| `error`   | [Transform](#transform) | Transform to be resolved in case of an error |

**Case Statements**

The `select` array may contain one or more case statements, similar to a `switch` in plain javascript. It will execute from top to bottom, until it finds a statement that the case results in a 'truthy' value. Once it finds a match it will execute its `do` transform to resolve the entity.

In the given case that no case statement is resolved to `truthy`, then the default statement will be used as the entity's resolution. 

**IMPORTANT:**

`default` is mandatory, if not provided hydra will throw an error when trying to parse the entity.

```js

case isEqual = (compareTo) => (ctx, next) => {
  next(null, ctx.value === compareTo)
}

case resolveTo = (value) => (ctx, next) => {
  next(null, value)
}

case throwError = (message) => (ctx, next) => {
  // next uses Node.js callback style, where first 
  // argument is error
  next(new Error(message))
}

hydra.addEntities({
  'control:fruitPrices': {
    select: [
      { case: isEqual('oranges') do: resolveTo(0.59) },
      { case: isEqual('apples') do: resolveTo(0.32) },
      { case: isEqual('bananas') do: resolveTo(0.48) },
      { case: isEqual('cherries') do: resolveTo(3.00) },
      { default: throwError('Fruit was not found!! maybe call the manager?') },
    ]
  }
})

hydra.transform('control:fruitPrices', 'apples').then((ctx) => {
  console.log(ctx.value) // 0.32
});

hydra.transform('control:fruitPrices', 'cherries').then((ctx) => {
  console.log(ctx.value) // 3.00 expensive!! 
});

hydra.transform('control:fruitPrices', 'plum')
.catch((error) => {
  console.log(error) // Fruit was not found!! maybe call the manager?
});
```

**EXAMPLES**

For examples of Control entities please look at the ones used on the unit tests: [Control Definitions](https://github.com/Viacom/hydra/blob/master/test/definitions/control.js), and [Integration Examples](https://github.com/Viacom/hydra/blob/master/test/definitions/integrations.js)

#### <a name="schema"></a> Schema

Run a JSON Schema against a data structure. Uses [Avj](https://github.com/epoberezkin/ajv) behind the scene.

**SYNOPSIS**

```js
{
  value: Transform,

  schema: JSONSchema,
  options: Object,

  before: Transform,
  after: Transform,
  error: Transform,
  params: Object
}
```

**Properties exposed:**

| Key | Type | Description |
|:---|:---|:---|
| `context` | [Transform](#transform) | The value this Entity will pass to the schema validation |
| `schema`   | `Object` | Valid [JSON Schema](http://json-schema.org/documentation.html) Object |
| `options` | `Object` | Avj's [options](https://github.com/epoberezkin/ajv#options) object
| `params`    | `Object` | User defined Hash that will be passed to every transform within the context of the Transform's execution |
| `before` | [Transform](#transform) | Transform to be resolved in **before** making the request |
| `after` | [Transform](#transform) | Transform to be resolved in **after** making the request |
| `error`   | [Transform](#transform) | Transform to be resolved in case of an error |

**EXAMPLES**

For examples of Schema entities please look at the ones used on the unit tests: [Schema Definitions](https://github.com/Viacom/hydra/blob/master/test/definitions/schema.js), and [Integration Examples](https://github.com/Viacom/hydra/blob/master/test/definitions/integrations.js)

#### <a name="entity-compose-reducer"></a> Entity Compose Reducer

This reducer is an `Array` of [ComposeReducer](#compose-reducer) Objects, each reducer gets executed asynchronously. You may add as many supported reducers as you want by the Entity. The result of one gets passed to the next reducer. 

<a name="compose-reducer"></a> A **ComposeReducer** Is an Object with a single 'key', the key is the type of reducer you want to execute, and the value is the body of the modifier.

**SYNOPSIS**

```js
...
compose: [
  { // ComposeReducer
    <reducerType>: <reducerBody>
  },
  ...
],
...
```

**EXAMPLES**

For examples of hash entity compose implementation look at [Hash Compose Example](https://github.com/Viacom/hydra/blob/master/examples/entity-hash-compose.js)

### <a name="extending-entities"></a>Extending Entities

You may extend entity definitions, this functionality is mainly with the purpose of DRY principle. It is not meant to work as an inheritance pattern, but more of overriding entity A with entity B. 

Extending entities is **not a deep merge of properties** from one entity to the other, it only overrides the first level own enumerable properties from base entity to target.

**SYNOPSIS**

```js
...
`entity:child -> entity:base`: { // read entity:child extends entity:base
  // entity spec
},
...
```

**EXAMPLE**

To reuse the options from an extension: 

```js
hydra.addEntities({
  'entry:getReposWithAllTags': {
    value: 'source:repositories'
  },
  'source:githubBase': {
    options: { headers: { 'User-Agent': 'hydra' } }
  },
  'source:repositories -> source:githubBase': {
    // options object is provided by source:githubBase
    url: 'https://api.github.com/orgs/{locals.orgName}/repos'
  }
})

const options = {
  locals: {
    orgName: 'nodejs'
  }
}

hydra.transform('entry:getReposWithAllTags', null, options).then((ctx) => {
  console.log(ctx.value) // returns all the repos for nodejs org
})
```

Example at: [examples/extend-entity-keys.js](examples/extend-entity-keys.js)


```js
hydra.addEntities({
  'hash:multiply': {
    mapKeys: {
      multiplyByFactor: '$multiplier | model:multiplyBy',
      multiplyBy20: '$multiplier | model:multiplyBy20'
    }
  },
  'model:multiplyBy': {
    value: (ctx, next) => next(null, ctx.value * ctx.params.multiplicand),
    params: {
      multiplicand: 1
    }
  },
  'model:multiplyBy20 -> model:multiplyBy': {
    // through the params property we can
    // parameterize the base entity
    params: {
      multiplicand: 20
    }
  }
})

hydra.transform('hash:multiply', {multiplier: 5}).then((ctx) => {
  console.log(ctx.value)
  /*
  {
    multiplyByFactor: 5,
    multiplyBy20: 100
  }
  */
})
```

Example at: [examples/extend-entity-reusability.js](examples/extend-entity-reusability.js)

### <a name="api-hydra-use"></a>hydra.use

Adds a middleware method. Middleware in Hydra context is a place to add logic that you want to be executed `before` and `after` the execution of all Entities hydra has registered.

This Middleware layer allows you to have control over the execution of Entities. 

**SYNOPSIS**

```js
hydra.use(id, callback)
```

**ARGUMENTS**

| Argument | Type | Description |
|:---|:---|:---|
| `id` | `string` | middleware id is a two part string in the form of `<EntityType>:<EventType>`. Where `<EntityType>` may be `entry`, `model`, `source`, or any other registered entity type. `<EventType>` is either `before` or `after`. |
| `callback` | `Function` | This is the callback function that will be executed once an entity event is triggered. The callback has the form `(acc, next)`, where `acc` is the current [middleware context object](#middleware-reducer-object), and next is a function callback to be executed once the middleware is done executing. `next` callback uses the form of `(error)`. |

### <a name="middleware-reducer-object"></a> Middleware context object

This Object is a copy of the currently executing Entity's [Accumulator](#accumulator) Object with a method `resolve(value)` appended to it. This method allows you to control the execution of the middleware chain. When/if `acc.resolve(value)` is called inside a middleware callback it will skip the complete execution of the Entity resolving to the given value. (FIXME: _need help rephrasing this section_).

**SYNOPSIS**

```js
{ // extends Accumulator
  resolve: Function,
  ...
}
```

**API:**

| Key | Type | Description |
|:---|:---|:---|
| `resolve` | `Function` | Will resolve the entire entity with the value passed, this function has the form of: `(value)` |

### <a name="api-hydra-add-value"></a>hydra.addValue

Adds values to later access them inside any [Accumulator](#accumulator).value.

**SYNOPSIS**

```js
hydra.addValue(objectPath, value)
```

**ARGUMENTS**

| Argument | Type | Description |
|:---|:---|:---|
| `objectPath` | `string` | object path were you wish to add the new value, it uses [_.set](https://lodash.com/docs/4.17.4#set) to append to the values object |
| `value` | `*` | anything you wish to store |


## Integration with Express

Below is an example of how you may create a data api by mapping entries to expose them to a routing system using ExpressJs. 

```js
const app = express()

const hydra = Hydra.create({ /* initialize hydra options */})

app.get('/data/:entry', (req, res, next) =>{
  const {entry} = req.params
  hydra.transform(`entry:${entry}`, req.query, (err, res) => {
    if (err) {
      console.error('entry: %s failed!', entry)
      console.error(error.stack)
      next(err) // pass error to middleware chain
      return
    }
    // respond with result from hydra
    res.send(result.value))
  }))
})
```
