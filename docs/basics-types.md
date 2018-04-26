---
id: basics-types
title: Type creation
---

## Scalar types
Graphql-compose has following built-in scalar types: `String`, `Float`, `Int`, `Boolean`, `ID`, `Date`, `JSON`.

## Object types via TypeComposer
If you need to create some complex type, you will need to use `TypeComposer`. It's a builder for `GraphQLObjectType` object.
`TypeComposer` helps to create and modify types before you build a schema. It has a bunch of very useful methods for writing your type generators. With `GraphQL.js` you may write Type configs once, with `graphql-compose` you may also may pass your Types via series of modification methods.

### Object type creation
`TypeComposer` has very convenient ways of type creation via `create` method.

#### ... without fields
Useful when you write your own type generators.
```js
const AuthorTC = TypeComposer.create('Author');
```

#### ... via config
Most recommended way to define your Type.
```js
const AuthorTC = TypeComposer.create({
  name: 'Author',
  fields: {
    id: 'Int!',
    firstName: 'String',
    lastName: 'String',
    posts: {
      type: () => PostTC, // wrapping with arrow function helps to solve hoisting problems
      description: 'Posts written by Author',
      resolve: (source, args, context, info) => { ... },
    },
  },
});
```

#### ... via SDL
```js
const AuthorTC = TypeComposer.create(`
  type Author {
    id: Int!
    firstName: String
    lastName: String
  }
`);
```

#### ... via GraphQLObjectType
This is very useful when you want modify existed `GraphQLObjectType`.
```js
const AuthorType = new GraphQLObjectType(...)
const AuthorTC = TypeComposer.create(AuthorType);
```

#### Creating TypeComposer from scratch

```js
import { TypeComposer } from 'graphql-compose';

// creating type User with 4 fields
export const UserTC = TypeComposer.create(`
  type User {
    name: String
    nickname: String
    views: Int
    # Field with any type of data
    someJsonMess: Json
  }
`);

// adding fifth field `tweets` with fetching from some remote API
UserTC.addField('tweets', {
  type: 'type TweetList { msg: String, createdAt: Date }',
  resolve: (source, args, context, info) =>
    fetch(`[api_endpoint]/${source.nickname}`).then(res => res.json()),
});

// Add resolveк - helper for fetching data
UserTC.addResolver({
  name: 'findById',
  args: { id: 'Int' },
  type: UserTC,
  resolve: async ({ source, args }) => {
    const res = await fetch(`/endpoint/${args.id}`); // or some fetch from any database
    const data = await res.json();
    // here you may clean up `data` response from API or Database,
    // it should has same shape like UserTC fields
    // eg. { name: 'Peter', nickname: 'peet', views: 20, someJsonMess: { ... } }
    // if some fields are undefined or missing, graphql return `null` on that fields
    return data;
  },
});

// If you need you may get generated GraphQLObjectType via UserTC.getType();
```

#### Converting a Mongoose model to TypeComposer (wrapped GraphQLObjectType) with `graphql-compose-mongoose`.

You may create TypeComposer from mongoose model with bunch of useful resolvers `findById`, `findMany`, `updateOne`, `removeMany` and so on. Read more about [graphql-compose-mongoose](https://github.com/nodkz/graphql-compose-mongoose)

```js
import mongoose from 'mongoose';
import composeWithMongoose from 'graphql-compose-mongoose';

const PersonSchema = new mongoose.Schema({
  firstName: { type: String },
  lastName: { type: String },
  email: { type: String },
});
export const Person = mongoose.model('Person', PersonSchema);

// here composeWithMongoose will create type with fields from mongoose schema
export const PersonTC = composeWithMongoose(Person);

// If you need you may get generated GraphQLObjectType via PersonTC.getType();
```

----------

Sooner or later you need to edit the types. There are functions to get and set one or more fields, aswell as you can get data from the fields too

### Adding fields

```js
UserTC.addFields({
  fullName: {
    type: 'String',
    resolve: (source) => `${source.firstName} ${source.lastName}`,
    projection: { firstName: true, lastName: true },
  }
});

UserTC.addFields({
  age: {
    type: 'Int',
    // other field config properties
  }
  ageShort: 'Int', // just type
  complex: {
    type: `type Address {
      city: String
      street: String
    }`,
    // other field config properties
  },
  complexShort: `type AddressShort {
    city: String
    street: String
  }`
});
```

### Adding relations with other TypeComposers

Assume that `ArticleTC` is generated by [graphql-compose-mongoose](https://github.com/nodkz/graphql-compose-mongoose). So let take generated `findMany` resolver from `ArticleTC` and connect/relate it with user data:

```js
UserTC.addRelation('last10Articles', {
  resolver: () => ArticleTC.getResolver('findMany'),
  prepareArgs: {
    filter: source => ({ userId: `${source._id}` }), // calculate `filter` argument
    limit: 10, // set value to `limit` argument
    sort: { _id: -1 }, // set `sort` argument
    skip: null, // remove `skip` argument
  },
  projection: { _id: true },
});
```

Read more about relations [here](04-relations.md).

### Remove fields

```js
UserTC.removeField('email');
```

**_REMEMBER: This will remove the field in all objects where this TC is used_**

### Field functions

```js
getFieldNames(): string[]
getField(fieldName: string): ?GraphQLFieldConfig
getFields(): GraphQLFieldConfigMap
setFields(fields: GraphQLFieldConfigMap): void
setField(fieldName: string, fieldConfig: GraphQLFieldConfig)
addFields(newFields: GraphQLFieldConfigMap): void
hasField(fieldName: string): boolean
removeField(fieldNameOrArray: string | Array<string>): void
removeOtherFields(keepFields: Array<string>); // will remove all other fields
getFieldType(fieldName: string): GraphQLOutputType | void
getFieldArgs(fieldName: string): ?GraphQLFieldConfigArgumentMap
getFieldArg(fieldName: string, argName: string): ?GraphQLArgumentConfig
extendField(fieldName: string, parialFieldConfig: GraphQLFieldConfig): GraphQLFieldConfig)
deprecateFields(fieldMap: { [fieldName:string]: deprecation reason string } );
```