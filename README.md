Yup
=======================

Yup is a js object schema validator. The api and style is heavily inspired by [Joi](https://github.com/hapijs/joi), which is an amazing library but generally too big and feature rich for general browser use. Yup is a leaner in the same spirit without the fancy features. You can use it on the server as well, but in that case you might as well just use Joi.

Yup is also a a good bit less opinionated than joi, allowing for custom validation and transformations. It also allows "stacking" conditions via `when` for properties that depend on more than one other sibling or child property.

## Changes 0.5.0

- __breaking__ `isValid` is now async, provide a node style callback, or use the promise the method returns to read the validity. This change allows 
for more robust validations, specifically remote ones for client code (or db queries for server code). The `cast` method is still, and will remain, synchronuous.
- added `validate` method (also async) which resolves to the value, and rejects with a new `ValidationError`

## Usage

You define and create schema objects. Schema objects are immutable, so each call of a method returns a _new_ schema object.
  
    var yup = require('yup')

    var schema = yup.object().shape({
      name:      yup.string().required(),
      age:       yup.number().required().positive().integer(),
      email:     yup.string().email(),
      website    yup.string().url(),
      createdOn: yup.date().default(function() { 
        return new Date 
      }),
    })

    //check validity
    schema.isValid({
      name: 'jimmy',
      age: 24
    })
    .then(function(valid){
      valid // => true  
    })  


    //you can try and type cast objects to the defined schema
    schema.cast({
      name: 'jimmy',
      age: '24',
      createdOn: '2014-09-23T19:25:25Z'
    })
    // => { name: 'jimmy', age: 24, createdOn: Date }

### `yup`

The module export.

```js
var yup = require('yup')

yup.mixed
yup.string
yup.number

yup.bool
yup.boolean
yup.date
yup.object
yup.array

yup.reach
```

### `.reach(Schema schema, String path, Object options)`

For nested schema's `yup.reach` will retrieve a nested schema based on the provided path.


### `mixed`

Creates a schema that matches all types. All types inherit from this base type

```javascript
var schema = yup.mixed();
schema.isValid(undefined, function(valid){
  valid //=> true
}) 
```

#### `mixed.clone()`

Creates a new instance of the schema. Clone is used internally to return a new schema with every schema state change. 

#### `mixed.concat(schema)`

Creates a new instance of the schema by combining two schemas.

#### `mixed.validate(value, [options, callback])`

Returns the value (a cast value if `isStrict` is `false`) if the value is valid, and returns the errors otherwise. This method is __asynchronous__ 
and returns a Promise object, that is fulfilled with the value, or rejected with a `ValidationError`. If you are more comfortable with 
Node style callbacks, then you can provide one to be called when the validation is complete (called with the Error as the first argument, and value 
as the second).

```javascript
schema.validate({ name: 'jimmy',age: 24 })
  .then(function(value){
    value // => { name: 'jimmy',age: 24 }  
  })

schema.validate({ name: 'jimmy', age: 'hi' })
  .catch(function(err){
    err.name   // 'ValidationError'
    err.errors // => ['age must be a number']
  })

//or with callbacks

schema.validate({ name: 'jimmy',age: 24 }, function(err, value){
  err === null // true
  value        // => { name: 'jimmy',age: 24 }  
})

schema.validate({ name: 'jimmy', age: 'hi' }, function(err, value){
  err.name   // 'ValidationError'
  err.errors // => ['age must be a number']
  value === undefined // true
})
```

#### `mixed.isValid(value, [options, callback])`

Returns `true` when the passed in value matches the schema. if `false` then the schema also has a `.errors` field which is an array of validation error messages (strings), thrown by the schema. `isValid` is __asynchronous__ and returns a Promise object. If you are more comfortable with 
Node style callbacks, then you can provide one to be called when the validation is complete.

The `options` argument is an object hash containing any schema options you may want to override (or specify for the first time).

- `strict` -> boolean: default `false`
- `context` -> an object hash containing any context for validating schema conditions (see: `when()`)

#### `mixed.cast(value)`

Attempts to coerce the passed in value to a value that matches the schema. For example: `'5'` will cast to `5` when using the `number()` type. Failed casts generally return `null`, but may also return results like `NaN` and unexpected strings.

#### `mixed.isType(value)`

Runs a type check against the passed in `value`. It returns true if it matches, it does not cast the value. When `nullable()` is set `null` is considered a valid value of the type.

#### `mixed.strict()` (default: `false`)

Sets the `strict` option to `true`, telling the schema to not try and cast the passed in value before validating it.

#### `mixed.default(value)`

Sets a default value to use when the value is `undefined`. The default value will be cloned on each use wich can incur performance penalty for objects and arrays. To avoid this overhead you can also pass a function that returns an new default.

```js
  yup.string.default('nothing');

  yup.object.default({ number: 5}); // object will be cloned every time a default is needed

  yup.object.default(() => ({ number: 5})); // this is cheaper

  yup.date.default(() => new Date()); //also helpful for defaults that change over time

```

#### `mixed.nullable(isNullable)` (default: `false`)

Indicates that `null` is a valid value for the schema. Without `nullable()` 
`null` is treated as a different type and will fail `isType()` checks.

#### `mixed.required(msg)`

Mark the schema as required. All field values apart from `undefined` meet this requirement.

#### `mixed.oneOf(arrayOfValues)` Alias: `equals`

Whitelist a set of values. Values added are automatically removed from any blacklist if they are in it.

```javascript
var schema = yup.mixed().oneOf(['jimmy', 42]);
schema.isValid(42)       //=> true
schema.isValid('jimmy')  //=> true
schema.isValid(new Date) //=> false
```

#### `mixed.notOneOf(arrayOfValues)`

Blacklist a set of values. Values added are automatically removed from any whitelist if they are in it.

```javascript
var schema = yup.mixed().notOneOf(['jimmy', 42]);
schema.isValid(42)       //=> false
schema.isValid(new Date) //=> true
```

#### `mixed.when(key, options | function)`

Adjust the schema based on a sibling or sibling children fields. You can provide an object literal where the key `is` is value or a matcher function, `then` provides the true schema and/or `otherwise` for the failure condition.

`is` conditions are strictly compared (`===`) if you want to use a different form of equality you can provide a function like: `is: (value) => value == true`.

Alternatively you can provide a function the returns a schema ( the `this` value is the current schema). `when` conditions are additive. 

```javascript
var inst = yup.object({
      isBig: yup.boolean(), 
      other: yup.number(),
      count: yup.number()
        .when('isBig', { 
          is: true,  // alternatively: (val) => val == true
          then:      yup.number().min(5), 
          otherwise: yup.number().min(0) 
        })
        .when('other', function(v){ 
          if (v === 4) return this.max(6)
        })
    })
```

__note: because `when` conditions must be resolved during `cast()`, a synchronous api, `is` cannot be a schema object as checking schema validity it is asynchronous__

#### `mixed.validation(message, fn, [callbackStyleAsync])`

Adds a validation function to the validation chain. Validations are run after any object is cast. Many types have some validations built in, but you can create custom ones easily. All validations are run asynchronously, as such their order cannot be guaranteed. The validation function should either return `true` or `false` directly, or return a promsie that resolves `true` or `false`. If you perfer the Node callback style, pass `true` for `callbackStyleAsync`  and the validation function will pass in an additional `done` function as the last parameter, which should be called with the validity.

for the `message` argument you can provide a string which is will interpolate certain keys if specified, all validations are given a `path` value which indicates location.

```javascript
var schema = yup.mixed().validation('${path} is invalid!', function(value){
  return value !== 'jimmy' //or return a Promise here
});

//or callback style
var schema = yup.mixed().validation('${path} is invalid!', function(value, done){
  done(null, value !== 'jimmy') //error is for exceptions, not an invalid value
}, true);

schema.isValid('jimmy') //=> true

schema.isValid('john') //=> false
schema.errors // => [ 'this is invalid!']
```

#### `mixed.transform(fn)`

Adds a transformation to the transform chain. Transformations are part of the casting process and run after the value is coerced to the type, but before validations. Transformations will not be applied unless `strict` is `true`. Some types have built in transformations. 

Transformations are useful for arbitrarily altering how the object is cast. You should take care not to mutate the passed in value if possible.

```javascript
var schema = yup.string().transform(function(value){
  return value.toUpperCase()
});
schema.cast('jimmy') //=> 'JIMMY'
```

#### Static Methods

- `Mixed.create(props)` - creates a new instance of a type with the specified props
- `Mixed.extend(protoProps)` - Backbone-esque object inheritance. extend returns a new constructor function that inherits from the type. All types inherit `mixed` in this manner. Be sure to include a `constructor` property it is not automatically created.


### string

Define a string schema. __note: strings are nullable by default.__ 

```javascript
var schema = yup.string();
schema.isValid('hello') //=> true
```

#### `string.min(limit, message)`

Set an minimum length limit for the string value. The `${min}` interpolation can be used in the `message` argument

#### `string.max(limit, message)`

Set an maximum length limit for the string value. The `${max}` interpolation can be used in the `message` argument

#### `string.matches(regex, message)`

Provide an arbitrary `regex` to match the value against.

```javascript
var v = string().matches(/(hi|bye)/);
v.isValid('hi').should.eventually.equal(true)
v.isValid('nope').should.eventually.equal(false)
```

#### `string.email(message)`

Validates the value as an email address via a regex.

#### `string.url(message)`

Validates the value as a valid URL via a regex.


#### `string.trim(msg)`

Transforms string values by removing leading and trailing whitespace. If 
`strict()` is set it will only validate that the value is trimmed.

#### `string.lowercase(msg)`

Transforms the string value to lowercase. If `strict()` is set it will only validate that the value is lowercase.

#### `string.uppercase(msg)`

Transforms the string value to uppercase. If `strict()` is set it will only validate that the value is uppercase.

### number

Define a number schema.

```javascript
var schema = yup.number();
schema.isValid(10) //=> true
```

#### `number.min(limit, message)`

Set the minimum value allowed. The `${min}` interpolation can be used in the 
`message` argument.

#### `number.max(limit, message)`

Set the maximum value allowed. The `${max}` interpolation can be used in the 
`message` argument.

#### `number.positive(message)`

Value must be a positive number.

#### `number.negative(message)`

Value mut be a negative number.

#### `number.integer(message)`

Transformation that coerces the value into an integer via truncation 
` value | 0`. If `strict()` is set it will only validate that the value is an integer.

#### `round(type)` - 'floor', 'ceil', 'round'

Rounds the value by the specified method (defaults to 'round').


### boolean

Define a boolean schema.

```javascript
var schema = yup.boolean();
schema.isValid(true) //=> true
```

### date

Define a Date schema. By default ISO date strings will parse correctly.

```javascript
var schema = yup.date();
schema.isValid(new Date) //=> true
```

#### `date.min(limit, message)`

Set the minimum date allowed.

#### `date.max(limit, message)`

Set the maximum date allowed.

### array

Define an array schema. Arrays can be typed or not, When specifying the element type, `cast` and `isValid` will apply to the elements as well. Options passed into `isValid` are passed also passed to child schemas.

```javascript
var schema = yup.array().of(number().min(2));
schema.isValid([2, 3])   //=> true
schema.isValid([1, -24]) //=> false

schema.cast(['2', '3'])  //=> [2, 3] 
```

### `array.of(type)`

Specify the schema of array elements. It can be any schemaType, and is not required.

#### `array.min(limit, message)`

Set an minimum length limit for the array. The `${min}` interpolation can be used in the `message` argument.

#### `array.max(limit, message)`

Set an maximum length limit for the array. The `${max}` interpolation can be used in the `message` argument.

### `array.compact(rejector)`

Removes falsey values from the array. Providing a rejector function lets you specify the rejection criteria yourself.

```javascript
array()
  .compact()
  .cast(['', 1, 0, 4, false, null]) // => [1,4]

array()
  .compact(function(v){ 
    return v == null 
  })
  .cast(['', 1, 0, 4, false, null]) // => ['',1, 0, 4, false]
```

### object

Define an object schema. Options passed into `isValid` are also passed to child schemas.

```javascript
yup.object().shape({
  name:      string().required(),
  age:       number().required().positive().integer(),
  email:     string().email(),
  website    string().url(),
}) 
```

#### `object.shape(schemaHash)`

Define the keys of the object and the schemas for said keys.

#### `object.from(fromKey, toKey, alias)`

Transforms the specified key to a new key. If `alias` is `true` then the old key will be left.

```javascript
var schema = object()
      .shape({
        myProp: mixed(),
        Other: mixed(),
      })
      .from('prop', 'myProp')
      .from('other', 'Other', true)

inst.cast({ prop: 5, other: 6}) // => { myProp: 5, other: 6, Other: 6 }
```

#### `object.camelcase()`

Transforms all object keys to camelCase

#### `object.constantcase()`

Transforms all object keys to CONSTANT_CASE.
