# Mongof

A practical functional library to use some of the key features that [mongodb](https://www.npmjs.com/package/mongodb) 
provide in functional programming way. 

Focus of the library is definitely not to provide all features that `mongodb` provides.
Instead, Mongof will provide you the most common functions wrapped with curried functions, 
so that you can skip initial setup for client and retrieval of db and collection objects,
and make them extremely reusable across your entire project.

Library is completely dependent on [ramda](https://github.com/ramda/ramda) library to
provide functional programming style for mongodb.

## Why Mongof?

It is simple and extremely extensible with the power of functional programming paradigm.
Since the most of the functions provided are curried functions, 
you can make your usage of these functions partially recomposable very easily.

See [this](https://fr.umio.us/favoring-curry/) for more info about currying.

## Installation

Simply run the following command:

`$ npm install mongof`

## Usage

Mongof essentially provides you some useful functions to connect a MongoDB client,
and perform most used operations on a MongoDB instance including executing query, 
and saving document. 

Also, Mongof provides `useDb` and `useCollection` functions
that simply gives you original results from `mongodb` for you to freely perform 
all kinds of `mongodb` operations that Mongof does not wrap yet.

### Connecting a MongoDB instance

Before you can perform operations on a MongoDB instance, first we need to connect to it.
To connect a MongoDB instance you can use `createClient` function.

```
const { createClient } = require('mongof');

const connectionString = 'mongodb://root:rootpassword@localhost:27017';
const options = {
  useNewUrlParser: true,
  useUnifiedTopology: true,
};

// Connecto given MongoDB instance and return a Promise<MongoClient>
// which you can do any configuration mongodb provides
createClient(connectionString, options).then(client => {
  const db = client.db('dbName');
  const collection = db.collection('collectionName');

  return collection.find({}).toArray();
}).then(console.log);

```

#### Memoization
`createClient` is a memoized functtion, so it will return the same instance
if you call it with the same arguments.

For more info: see 
[memoizeWith](https://ramdajs.com/docs/#memoizeWith) 
and [Memoization (1D, 2D and 3D)](https://www.geeksforgeeks.org/memoization-1d-2d-and-3d/)

All other main functions of Mongof requires `client`

### useDb and useCollection

In `mongodb`, we start with connection client and pass a callback function to perform
operations over connected `mongodb` instance. It is repeated and very imperative way of
using mongodb. In Mongof, you can use `useDb` and `useCollection` functions which
accepts `client` as first argument to use with, to perform all kinds of `Db` and `Collection` 
in declarative way.

```
// useMainDb.js
const { createClient, useDb } = require('mongof');

const client = createClient(connectionString);

export const useMainDb = () => useDb(client, 'mainDb');

// someOther.js
const { useMainDb } = require('./useMainDb');

useMainDb().then(console.log); // Original Db object of mongodb

```

Ok, I can hear you saying "Where is the functional programming?". 

Functional programming actually starts with `useCollection`. 
However, before we dive into functional programming realm, let's first understand
the basic usage of the function.

Let's make an example of fetching all data that a collection contains.
```
const { createClient, useCollection } = require('mongof');

const client = createClient(connectionString);

useCollection(client, 'mainDb', 'someCollection')
    .then(collection => {
            return collection.find({}).toArray(); // retrieve all
        }
    )
    .then(console.log);
```

Since it uses original `Collection` object of `mongodb`, you can simply
do anything you want that `mongodb` provides.

If we want to make it reusable across our entire app we can benefit from
currying.

As we noted before, most of the functions are curried in Mongof.

Let's make an example and compose a function which will help you use collections
in a db without repeating code for set-up.

```
const { createClient, useCollection } = require('mongof');

const client = createClient(connectionString);

// Here we do not pass the last argument so that it will return another
// function which accepts only collectionName argument
const useCollectionInMainDb = useCollection(client, 'mainDb');

useCollectionInMainDb('categories').then(console.log);
useCollectionInMainDb('articles').then(console.log);
useCollectionInMainDb('users').then(console.log);
```

### __ object

Before we give you more info about CRUD operations in Mongof, we need to understand
one crucial object `__`. It is a special placeholder which you can use in curried functions
to be able to recompose your function allowing partial application of any combination
 f arguments, regardless of their positions.
 
```
const { createClient, useCollection } = require('mongof');

const client = createClient(connectionString);

const useCategoriesIn = useCollection(client, __, 'categories');

useCategoriesIn('mainDb').then(console.log);
useCategoriesIn('otherDb').then(console.log);
```

As you may have noticed, after passing `__` object to useCollection as `dbName` argument,
`useCollection` returns another function which you can pass any 'dbName' you need, so you
reuse the rest of the setup of the function. This is useful in mostly CRUD operations since
they await more parameters from you to perform certain operations.

### Using findBy, findAll, findById, and findByObjectId

`useDb` and `useCollection` functions provide full capability of
using original `Db` and `Collection` objects in `mongodb` in functional way.
However, we tend to use `mongodb` for mostly CRUD operations.

Mongof provide following functions for finding operations:
- `findBy(client, dbName, collectionName, predicate) : Promise<Array>`
    - Accepts predicate object as it is documented at 
    [here](http://mongodb.github.io/node-mongodb-native/3.5/reference/ecmascriptnext/crud/#read-methods)
    and returns a `Promise<Array>`.
- `findAll(client, dbName, collectionName) : Promise<Array>`
    - Does not need any additional specific argument. 
    It simply returns all data in given collection as `Promise<Array>`
- `findById(client, dbName, collectionName, id) : Promise<object>`
    - Accepts `id` value to look for the `Document` with given `id` in given
    collection, and return `Promise<object>`.
- `findByObjectId(client, dbName, collectionName, id) : Promise<object>`
    - It's a wrapper function of `findById` to rescue you from repeatedly pass
    your id value in an instance of `ObjectId`.
    
All the CRUD operations functions are curried, so you can freely use them as we
used `useDb` and `useCollection` functions.

Here are some examples of usage.

```
// findBy example
const { createClient, findBy, __ } = require('mongof');

const client = createClient(connectionString);

const findInMaindDbBy = findBy(client, 'mainDb');

findInMaindDbBy('categories', { name: 'some Categery' }).then(console.log);
findInMaindDbBy('articles', { name: 'some Categery' }).then(console.log);

const findInCategoriesBy = findInMaindDbBy('categories');
findInCategoriesBy({ name: 'some Categery' }).then(console.log);
findInCategoriesBy({ description: 'some Categery description' }).then(console.log);

// Say we have multiple replica client
const client1 = createClient(connectionString, options);
const client2 = createClient(connectionString, options);

const findInMainDbUsing = findBy(__, 'mainDb');
const findInCategoriesUsing = (__, 'categories');

findInCategoriesUsing(client1, { name: 'some Categery' }).then(console.log);
findInCategoriesUsing(client2, { name: 'some Categery' }).then(console.log);

findInMainDbUsing(client1, 'users', { name: 'some User name' }).then(console.log);
findInMainDbUsing(client2, 'users', { name: 'some User name' }).then(console.log);

// You can think of all combinations you need. 
```

## TODO

- Implement insertOne
- Implement updateOne
- Make sure connections are closed after using any of these functions
- Tests