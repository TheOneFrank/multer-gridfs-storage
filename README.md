# Multer's GridFS storage engine

[![Build Status][travis-image]][travis-url] [![Coverage Status][coveralls-image]][coveralls-url] ![Npm version][version-image] ![Downloads][downloads-image]

[GridFS](https://docs.mongodb.com/manual/core/gridfs) storage engine for [Multer](https://github.com/expressjs/multer) to store uploaded files directly to MongoDb.

This module is intended to be used with the v1.x branch of Multer.

## Features

- Compatibility with MongoDb versions 2 and 3.
- Full Node.js support for versions 0.10 and greater.
- Really simple api.
- Caching of url based connections.
- Promise support.
- Generator function support.
- Support for existing and promise based database connections.
- Delayed file storage until the connection is available.

## Installation

Using npm

```sh
$ npm install multer-gridfs-storage --save
```

Basic usage example:

```javascript
const express = require('express');
const multer  = require('multer');

// Create a storage object with a given configuration
const storage = require('multer-gridfs-storage')({
   url: 'mongodb://yourhost:27017/database'
});

// Set multer storage engine to the newly created object
const upload = multer({ storage: storage });

const app = express()

// Upload your files as usual
const sUpload = upload.single('avatar');
app.post('/profile', sUpload, (req, res, next) => { 
    /*....*/ 
})

const arrUpload = upload.array('photos', 12);
app.post('/photos/upload', arrUpload, (req, res, next) => {
    /*....*/ 
})

const fUpload = upload.fields([{ name: 'avatar', maxCount: 1 }, { name: 'gallery', maxCount: 8 }])
app.post('/cool-profile', fUpload, (req, res, next) => {
    /*....*/ 
})
```

## API

### module(configuration): function

The module returns a function that can be invoked to create a Multer storage engine.

Check the [wiki][wiki] for an in depth guide on how to use this module.

### Configuration

The configuration parameter is an object with the following properties.

#### url

Type: `string`

Required if [`db`][db-option] option is not present

An url pointing to the database used to store the incoming files.
 
With this option the module will create a mongodb connection for you. It must be a standard mongodb [connection string][connection-string].

If the [`db`][db-option] option is specified this setting is ignored.

Example:

```javascript
const storage = require('multer-gridfs-storage')({
    url: 'mongodb://yourhost:27017/database'
});
```

> The connected database is available in the `storage.db` property.

> On mongodb v3 the client instance is also available in the `storage.client` property.

#### connectionOpts

Type: object

Not required

This setting allows you to customize how this module establishes the connection if you are using the [`url`][url-option] option. 

You can set this to an object like is specified in the [`MongoClient.connect`][mongoclient-connect] documentation and change the default behavior without having to create the connection yourself using the [`db`][db-option] option.

#### cache

Type: `boolean` or `string`

Not required

Default value: `false`

> This option only applies when you use an url string to connect to MongoDb. Caching is not enabled when you create instances with a [database][db-option] object directly.

Store this connection in the internal cache. You can also use a string to use a named cache. By default caching is disabled. See [caching](#caching) to learn more about reusing connections.

#### db

Type: [`DB`][mongo-db] or `Promise`

Required if [`url`][url-option] option is not present

The database connection to use or a promise that resolves with the connection.

This is useful to reuse an existing connection to create more storage objects.

Example:

```javascript
// mongodb v2 using a database instance
MongoClient.connect('mongodb://yourhost:27017/database').then(database => {
  storage = new GridFSStorage({ db: database });
});
```

```javascript
// mongodb v2 using a promise
const promise = MongoClient.connect('mongodb://yourhost:27017/database');
storage = new GridFSStorage({ db: promise });
```

```javascript
// mongodb v3 using a database instance
MongoClient.connect('mongodb://yourhost:27017').then(client => {
  const database = client.db('database')
  storage = new GridFSStorage({ db: database });
});
```

```javascript
// mongodb v3 using a promise
const promise = MongoClient
  .connect('mongodb://yourhost:27017')
  .then(client => client.db('database'));
  
storage = new GridFSStorage({ db: promise });
```

#### file

Type: `function` or `function*`

Not required

A function to control the file storage in the database. Is invoked **per file** with the parameters `req` and `file`, in that order.

By default, this module behaves exactly like the default Multer disk storage does. It generates a 16 bytes long name in hexadecimal format with no extension for the file to guarantee that there are very low probabilities of naming collisions. You can override this by passing your own function.

The return value of this function is an object or a promise that resolves to an object (this also applies to generators) with the following properties. 

Property name | Description
------------- | -----------
`filename` | The desired filename for the file (default: 16 byte hex name without extension)
`id` | An ObjectID to use as identifier (default: auto-generated)
`metadata` | The metadata for the file (default: `null`)
`chunkSize` | The size of file chunks in bytes (default: 261120)
`bucketName` | The GridFs collection to store the file (default: `fs`)
`contentType` | The content type for the file (default: inferred from the request)

Any missing properties will use the defaults.

If you return `null` or `undefined` from the file function, the values for the current file will also be the defaults. This is useful when you want to conditionally change some files while leaving others untouched.

This example will use the collection `'photos'` only for incoming files whose reported mime-type is `image/jpeg`, the others will be stored using default values.

```javascript
const GridFsStorage = require('multer-gridfs-storage');

const storage = new GridFsStorage({
  url: 'mongodb://host:27017/database',
  file: (req, file) => {
    if (file.mimetype === 'image/jpeg') {
      return {
        bucketName: 'photos'
      };
    } else {
      return null;
    }
  }
});
const upload = multer({ storage });
```

This other example names every file something like `'file_1504287812377'`, using the date to change the number and to generate unique values

```javascript
const GridFsStorage = require('multer-gridfs-storage');

const storage = new GridFsStorage({
  url: 'mongodb://host:27017/database',
  file: (req, file) => {
    return {
      filename: 'file_' + Date.now()
    };
  }
});
const upload = multer({ storage });
```

Is also possible to return values other than objects, like strings or numbers, in which case they will be used as the filename and the remaining properties will use the defaults. This is a simplified version of a previous example

```javascript
const GridFsStorage = require('multer-gridfs-storage');

const storage = new GridFsStorage({
  url: 'mongodb://host:27017/database',
  file: (req, file) => {
    // instead of an object a string is returned
    return 'file_' + Date.now();
  }
});
const upload = multer({ storage });
```

Internally the function `crypto.randomBytes` is used to generate names. In this example, files are named using the same format plus the extension as received from the client, also changing the collection where to store files to `uploads`

```javascript
const crypto = require('crypto');
const path = require('path');
const GridFsStorage = require('multer-gridfs-storage');

var storage = new GridFsStorage({
  url: 'mongodb://host:27017/database',
  file: (req, file) => {
    return new Promise((resolve, reject) => {
      crypto.randomBytes(16, (err, buf) => {
        if (err) {
          return reject(err);
        }
        const filename = buf.toString('hex') + path.extname(file.originalname));
        const fileInfo = {
          filename: filename,
          bucketName: 'uploads'
        };
        resolve(fileInfo);
      });
    });
  }
});
const upload = multer({ storage });
```

### File information

Each file in `req.file` and `req.files` contain the following properties in addition to the ones that Multer create by default. Most of them can be set using the [`file`][file-option] configuration.

Key | Description
--- | -----------
`filename` | The name of the file within the database
`metadata` | The stored metadata of the file
`id` | The id of the stored file
`bucketName` | The name of the GridFs collection used to store the file
`chunkSize` | The size of file chunks used to store the file
`size` | The final size of the file in bytes
`md5` | The md5 hash of the file
`contentType` | Content type of the file in the database
`uploadDate` | The timestamp when the file was uploaded

To see all the other properties of the file object, check the Multer's [documentation](https://github.com/expressjs/multer#file-information).

> Do not confuse `contentType` with Multer's `mimetype`. The first is the value in the database while the latter is the value in the request. You could choose to override the value at the moment of storing the file. In most cases both values should be equal. 

### Caching

You can enable caching by either using a boolean or a non empty string in the [cache][cache-option] option, then, when the module is invoked again with the same [url][url-option] it will use the stored db instance instead of creating a new one.

The cache is not a simple object hash. It supports handling asynchronous connections. You could, for example, synchronously create two storage instances one after the other and only one of them will try to open a connection. 

This greatly simplifies managing instances in different files of your app. All you have to do now is to store a url string in a configuration file to share the same connection. Scaling your application with a load-balancer, for example, can lead to spawn a great number of database connection for each child process. With this feature no additional code is required to keep opened connections to the exact number you want without any effort.

You can also create named caches by using a string instead of a boolean value. In those cases the module will uniquely identify the cache allowing for an arbitrary number of cached connections per url and giving you the ability to decide which connection to use and how many of them should be created. 

The following code will create a new connection and store it under a cache named `'default'`.

```javascript
const GridFsStorage = require('multer-gridfs-storage');

const storage = new GridFsStorage({
    url: 'mongodb://yourhost:27017/database',
    cache: true
});
```

Other, more complex example, could be creating several files and only two connections to handle them.

```javascript
 // file 1
const GridFsStorage = require('multer-gridfs-storage');

const storage = new GridFsStorage({
   url: 'mongodb://yourhost:27017/database',
   cache: '1'
});

// file 2
const GridFsStorage = require('multer-gridfs-storage');

const storage = new GridFsStorage({
    url: 'mongodb://yourhost:27017/database',
    cache: '1'
});

 // file 3
const GridFsStorage = require('multer-gridfs-storage');

const storage = new GridFsStorage({
   url: 'mongodb://yourhost:27017/database',
   cache: '2'
});

// file 4
const GridFsStorage = require('multer-gridfs-storage');

const storage = new GridFsStorage({
    url: 'mongodb://yourhost:27017/database',
    cache: '2'
});
```

The files 1 and 2 will use the connection cached under the key `'1'` and the files 3 and 4 will use the cache named `'2'`. You don't have to worry for managing connections anymore. By setting a simple string value the module manages them for you automatically.

Connection strings are parsed and tested for similarities. In this example the urls are equivalent and only one connection will be created.

```javascript
const GridFsStorage = require('multer-gridfs-storage');

// Both configurations are equivalent

const storage1 = new GridFsStorage({
    url: 'mongodb://host1:27017,host2:27017/database',
    cache: 'connections'
});

const storage2 = new GridFsStorage({
    url: 'mongodb://host2:27017,host1:27017/database',
    cache: 'connections'
});
```

Of course if you want to create more connections this is still possible. Caching is disabled by default so setting a `cache: false` or not setting any cache configuration at all will cause the module to ignore caching and create a new connection each time.

Using [connectionOpts][connectionOpts-option] has a particular side effect. The cache will spawn more connections only **when they differ in their values**. Objects provided here are not compared by reference as long as they are just plain objects. Falsey values like `null` and `undefined` are considered equal. This is required because various options can lead to completely different connections, for example when using replicas or server configurations. Only connections that are *semantically equivalent* are considered equal.

### Events

Each storage object is also a standard Node.js Event Emitter. This is done to ensure that some internal events can also be handled in user code.

#### Event: `'connection'`

This event is emitted when the MongoDb connection is ready to use.

*Event arguments*

 - db: The MongoDb database object that holds the connection

This event is triggered at most once.

#### Event: `'connectionFailed'`

This event is emitted when the connection could not be opened.

 - err: The connection error

This event only triggers at most once. 

> Only one of the events `connection` or `connectionFailed ` will be emitted.

#### Event: `'file'`

This event is emitted every time a new file is stored in the db. 

*Event arguments*

 - file: The uploaded file


#### Event: `'streamError'`

This event is emitted when there is an error streaming the file to the database.

*Event arguments*

 - error: The streaming error
 - conf: The failed file configuration
 
> Previously this event was named `error` but in Node `error` events are special and crash the process if one is emitted and there is no listener attached. You could choose to handle errors in an [express middleware][error-handling] forcing you to set an empty `error` listener to avoid crashing. To simplify the issue this event was renamed to allow you to choose the best way to handle storage errors.
 
#### Event: `'dbError'`
 
This event is emitted when the underlying connection emits an error.
 
 > Only available when the storage is created with the [`url`][url-option] option.
 
*Event arguments*
 
 - error: The error emitted by the database connection


## Test

To run the test suite, first install the dependencies, then run `npm test`:

```bash
$ npm install
$ npm test
```

Tests are written with [mocha](https://mochajs.org/) and [chai](http://chaijs.com/).

Code coverage thanks to [istanbul](https://github.com/gotwarlost/istanbul)

```bash
$ npm coverage
```

## License

[MIT](https://github.com/devconcept/multer-gridfs-storage/blob/master/LICENSE)

[travis-url]: https://travis-ci.org/devconcept/multer-gridfs-storage
[travis-image]: https://travis-ci.org/devconcept/multer-gridfs-storage.svg?branch=master "Build status"
[coveralls-url]: https://coveralls.io/github/devconcept/multer-gridfs-storage?branch=master
[coveralls-image]: https://coveralls.io/repos/github/devconcept/multer-gridfs-storage/badge.svg?branch=master "Coverage report"
[version-image]:https://img.shields.io/npm/v/multer-gridfs-storage.svg "Npm version"
[downloads-image]: https://img.shields.io/npm/dm/multer-gridfs-storage.svg "Monthly downloads"

[connection-string]: https://docs.mongodb.com/manual/reference/connection-string
[mongoclient-connect]: https://mongodb.github.io/node-mongodb-native/api-generated/mongoclient.html
[mongo-db]: https://mongodb.github.io/node-mongodb-native/api-generated/db.html
[error-handling]: https://github.com/expressjs/multer#error-handling 

[url-option]: #url
[connectionOpts-option]: #connectionOpts
[db-option]: #db
[file-option]: #file
[cache-option]: #cache
[wiki]: https://github.com/devconcept/multer-gridfs-storage/wiki
