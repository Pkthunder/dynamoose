# Schema

## new dynamoose.Schema(schema[, options])

You can use this method to create a schema. The `schema` parameter is an object defining your schema, each value should be a type or object defining the type with additional settings (listed below).

The `options` parameter is an optional object with the following options:

| Name | Type | Default | Information
|---|---|---|---|
| `saveUnknown` | array \| boolean | false | This setting lets you specify if the schema should allow properties not defined in the schema. If you pass `true` in for this option all unknown properties will be allowed. If you pass in an array of strings, only properties that are included in that array will be allowed. If you pass in an array of strings, you can use `*` to indicate a wildcard nested property one level deep, or `**` to indicate a wildcard nested property infinite levels deep. If you retrieve items from DynamoDB with `saveUnknown` enabled, all custom Dynamoose types will be returned as the underlying DynamoDB type (ex. Dates will be returned as a Number representing number of milliseconds since Jan 1 1970).
| `timestamps` | boolean \| object | false | This setting lets you indicate to Dynamoose that you would like it to handle storing timestamps in your documents for both creation and most recent update times. If you pass in an object for this setting you must specify two keys `createdAt` & `updatedAt`, each with a value of a string being the name of the attribute for each timestamp. If you pass in `null` for either of those keys that specific timestamp won't be added to the schema. If you set this option to `true` it will use the default attribute names of `createdAt` & `updatedAt`.

```js
const schema = new dynamoose.Schema({
	"id": String,
	"age": Number
}, {
	"saveUnknown": true,
	"timestamps": true
});
```

```js
const schema = new dynamoose.Schema({
	"id": String,
	"age": {
		"type": Number,
		"default": 5
	}
});
```

```js
const schema = new dynamoose.Schema({
	"id": String,
	"name": String
}, {
	"timestamps": {
		"createdAt": "createDate",
		"updatedAt": null // updatedAt will not be stored as part of the timestamp
	}
});
```

## Attribute Types

| Type | Set Allowed | DynamoDB Type | Custom Dynamoose Type | Nested Type | Settings | Notes |
|---|---|---|---|---|---|---|
| String | True | S | False | False |   |   |
| Boolean | False | BOOL | False | False |   |   |
| Number | True | N | False | False |   |   |
| Buffer | True | B | False | False |   |   |
| Date | True | N | True | False | **storage** - miliseconds \| seconds (default: miliseconds) | Will be stored in DynamoDB as milliseconds since Jan 1 1970, and converted to/from a Date instance. |
| Object | False | M | False | True |   |   |
| Array | False | L | False | True |   |   |

If you use a set you will define the type surrounded by brackets. For example a String Set would be defined as a type of `[String]`. Set's are different from Array's since they require each item in the Set be unique. If you use a Set, it will use the underlying JavaScript Set instance as opposed to an Array.

When using `saveUnknown` with a set, the type recognized by Dynamoose will be the underlying JavaScript Set constructor. If you have a set type defined in your schema the underlying type will be an Array.

Custom Dynamoose Types are not supported with the `saveUnknown` property. For example, if you wish you retrieve a document with a Date type, Dynamoose will return it as a number if that property does not exist in the schema and `saveUnknown` is enabled for that given property.

For types that are `Nested Types`, you must define a `schema` setting that includes the nested schema for that given attribute.

## Attribute Settings

### type: type | object

The type attribute can either be a type (ex. `Object`, `Number`, etc.) or an object that has additional information for the type. In the event you set it as an object you must pass in a `value` for the type, and can optionally pass in a `settings` object.

```js
{
	"address": {
		"type": Object
	}
}
```

```js
{
	"deletedAt": {
		"type": {
			"value": Date,
			"settings": {
				"storage": "seconds" // Default: miliseconds (as shown above)
			}
		}
	}
}
```

### schema: object | array

This property is only used for the `Object` or `Array` attribute types. It is used to define the schema for the underlying nested type. For `Array` attribute types, this value must be an `Array` with one element defining the schema. This element for `Array` attribute types can either be another raw Dynamoose type (ex. `String`), or an object defining a more detailed schema for the `Array` elements. For `Object` attribute types this value must be an object defining the schema. Some examples of this property in action can be found below.

```js
{
	"address": {
		"type": Object,
		"schema": {
			"zip": Number,
			"country": {
				"type": String,
				"required": true
			}
		}
	}
}
```

```js
{
	"friends": {
		"type": Array,
		"schema": [String]
	}
}
```

```js
{
	"friends": {
		"type": Array,
		"schema": [{
			"type": Object,
			"schema": {
				"zip": Number,
				"country": {
					"type": String,
					"required": true
				}
			}
		}]
	}
}
```

### default: value | function | async function

You can set a default value for an attribute that will be applied upon save if the given attribute value is `null` or `undefined`. The value for the default property can either be a value or a function that will be executed when needed that should return the default value. By default there is no default value for attributes.

```js
{
	"age": {
		"type": Number,
		"default": 5
	}
}
```

```js
{
	"age": {
		"type": Number,
		"default": () => 5
	}
}
```

You can also pass in async functions or a function that returns a promise to the default property and Dynamoose will take care of waiting for the promise to resolve before saving the object.

```js
{
	"age": {
		"type": Number,
		"default": async () => {
			const networkResponse = await axios("https://myurl.com/config.json").data;
			return networkResponse.defaults.age;
		}
	}
}
```

```js
{
	"age": {
		"type": Number,
		"default": () => {
			return new Promise((resolve) => {
				setTimeout(() => resolve(5), 1000);
			});
		}
	}
}
```

### forceDefault: boolean

You can set this property to always use the `default` value, even if a value is already set. This can be used for data that will be used as sort or secondary indexes. The default for this property is false.

```js
{
	"age": {
		"type": Number,
		"default": 5,
		"forceDefault": true
	}
}
```

### validate: value | RegExp | function | async function

You can set a validation on an attribute to ensure the value passes a given validation before saving the document. In the event you set this to be a function or async function, Dynamoose will pass in the value for you to validate as the parameter to your function. Validation will only be run if the item exists in the document. If you'd like to force validation to be run every time (even if the attribute doesn't exist in the document) you can enable `required`.

```js
{
	"age": {
		"type": Number,
		"validate": 5 // Any object that is saved must have the `age` property === to 5
	}
}
```

```js
{
	"id": {
		"type": String,
		"validate": /ID_.+/gu // Any object that is saved must have the `id` property start with `ID_` and have at least 1 character after it
	}
}
```

```js
{
	"age": {
		"type": String,
		"validate": (val) => val > 0 && val < 100 // Any object that is saved must have the `age` property be greater than 0 and less than 100
	}
}
```

```js
{
	"email": {
		"type": String,
		"validate": async (val) => {
			const networkRequest = await axios(`https://emailvalidator.com/${val}`);
			return networkRequest.data.isValid;
		} // Any object that is saved will call this function and run the network request with `val` equal to the value set for the `email` property, and only allow the document to be saved if the `isValid` property in the response is true
	}
}
```

### required: boolean

You can set an attribute to be required when saving documents to DynamoDB. By default this setting is `false`.

In the event the parent object is undefined and `required` is set to `false` on that parent attribute, the required check will not be run on child attributes.

```js
{
	"email": {
		"type": String,
		"required": true
	}
}
```

```js
{
	"data": {
		"type": Object,
		"schema": {
			"name": {
				"type": String,
				"required": true // Required will only be checked if `data` exists and is not undefined
			}
		}
		"required": false
	}
}
```

### enum: array

You can set an attribute to have an enum array, which means it must match one of the values specified in the enum array. By default this setting is undefined and not set to anything.

This property is not a replacement for `required`. If the value is undefined or null, the enum will not be checked. If you want to require the property and also have an `enum` you must use both `enum` & `required`.

```js
{
	"name": {
		"type": String,
		"enum": ["Tom", "Tim"] // `name` must always equal "Tom" or "Tim"
	}
}
```

### get: function | async function

You can use a get function on an attribute to be run whenever retrieving a document from DynamoDB. This function will only be run if the item exists in the document. Dynamoose will pass the DynamoDB value into this function and you must return the new value that you want Dynamoose to return to the application.

```js
{
	"id": {
		"type": String,
		"get": (value) => `applicationid-${value}` // This will prepend `applicationid-` to all values for this attribute when returning from the database
	}
}
```

### set: function | async function

You can use a set function on an attribute to be run whenever saving a document to DynamoDB. This function will only be run if the item exists in the document. Dynamoose will pass the value you provide into this function and you must return the new value that you want Dynamoose to save to DynamoDB.

```js
{
	"name": {
		"type": String,
		"set": (value) => `${value.charAt(0).toUpperCase()}${value.slice(1)}` // Capitalize first letter of name when saving to database
	}
}
```

### index: boolean | object | array

You can define indexes on properties to be created or updated upon model initialization. If you pass in an array for the value of this setting it must be an array of index objects. By default no indexes are specified on the attribute.

Your index object can contain the following properties:

| Name | Type | Default | Notes |
|---|---|---|---|
| name | string | `${attribute}${global ? "GlobalIndex" : "LocalIndex"}` | Name of index |
| global | boolean | false | If the index should be a global secondary index or not. Attribute will be the hash key for the index. |
| rangeKey | string | undefined | The range key attribute name for a global secondary index. |
| project | boolean \| [string] | true | Sets the attributes to be projected for the index. `true` projects all attributes, `false` projects only the key attributes, and an array of strings projects the attributes listed. |
| throughput | number \| {read: number, write: number} | undefined | Sets the throughput for the global secondary index. |


If you set `index` to `true`, it will create an index with all of the default settings.

```js
{
	"email": {
		"type": String,
		"index": {
			"name": "emailIndex",
			"global": true
		} // creates a global index with the name `emailIndex`
	}
}
```

```js
{
	"email": {
		"type": String,
		"index": {
			"name": "emailIndex",
			"throughput": {"read": 5, "write": 10}
		} // creates a global index with the name `emailIndex`
	}
}
```

### hashKey: boolean

You can set this to true to overwrite what the `hashKey` for the Model will be. By default the `hashKey` will be the first key in the Schema object.

```js
{
	"id": String,
	"key": {
		"type": String,
		"hashKey": true
	}
}
```

### rangeKey: boolean

You can set this to true to overwrite what the `rangeKey` for the Model will be. By default the `rangeKey` won't exist.

```js
{
	"id": String,
	"email": {
		"type": String,
		"rangeKey": true
	}
}
```
