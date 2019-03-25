# express-mong

Mongoose wrapper to inject request object into mongoose Query, Document, and Hooks

## Installation
```sh
$ npm install express-mong
```

## Introduction

express-mong is a very thin layer of mongoose 'Model' and 'Query'.
It enables to have request object in almost anywhere you access mongoose instances within that request scope,
and also provides same interfaces as mongoose 'Model' does including callbacks.

## Places you can find request object

You can find request object in:
- Document

- Document middleware
-- validate
-- save
-- remove

- Query middleware
-- count
-- deleteMany
-- deleteOne
-- find
-- findOne
-- findOneAndDelete
-- findOneAndRemove
-- findOneAndUpdate
-- update
-- updateOne
-- updateMany
-- replaceOne

- Aggregate middleware

- Methods
- Statics
- toObject
- toJSON

[Mongoose Middleware](https://mongoosejs.com/docs/middleware.html)

## Usage
```js
const express = require('express');
const Mong = require('express-mong');
const app = express();

app.use(Mong());

const testSchema = new Schema({ name: String, reqPath: String });

testSchema.pre('save', function(next) {
  const req = this.$req; // find request object from $req in document
  this.reqPath = req.path;
  next();
});

testSchema.pre('find', function() {
  const req = this.$req; // find request object from $req in query instance
  //
  // if session user is defined in request, can use it to limit the result
  // const userId = req.user._id;
  // const query = this.getQuery();
  // query.createdBy = userId;
  // this.setQuery(query);
  //
});

mongoose.model('Test', testSchema);

app.get('/test', function(req, res, next) {
  const Test = req.mong('Test');
  
  const test = new Test({ name: 'test1' });
  test.save(function(err, doc) {
    const _req = doc.$req; // find request object from $req in document
    res.send(doc.reqPath) // send '/test' set from save pre-hook
  });
});
```

## Document Decorator

You can set decorators for each mongoose schema and expect it to be run before document(s) been returned.

```js
...

Mong.setDecorator('Test', function() {
  this.name += this.name;
  return this;
});

app.get('/test2', function(req, res, next) {
  Test.findOne({ name: 'test1' }).then(doc => {
    res.send(doc.name); // send 'test1test1' set from decorator
  });
});
```

You can also set more than one decorators in options when initializing express middleware.

```js
...

app.use(Mong({
  decorators: {
    Test: function() {
      this.name += this.name;
      return this;
    },
    ...
  }
}));
```

Decorators can be either sync functions or async functions.

## Original Document Object

By default, the wrapper injects the original document object into returning documents.
The $original is an object converted by 'toObject' method of the original document.

```js
...
app.get('/test3', function(req, res, next) {
  const data = { name: 'test2' };
  Test.findOne({ name: 'test1' }).then(doc => {
    doc.set(data);
    console.log(doc.name); // test2
    console.log(doc.$original.name); // test1
    
    res.send(doc.$original.name); // send 'test1' set from original document object
  });
});
```
You can disable it in the options.

```js
...

app.use(Mong({
  keepOriginal: false,
  ...
}));
```

## Audit plugin

express-mong provides one mongoose plugin to audit user and datetime on document's creation and updates.
The user document / object must be set in request object as 'req.user'.

```js
const mongAudit = require('express-mong/plugins/audit');
...

const testSchema = new Schema({ name: String, reqPath: String });
testSchema.plugin(mongAudit);
mongoose.model('Test', testSchema);

app.get('/test', function(req, res, next) {
  console.log(req.user); // session user

  const Test = req.mong('Test');

  Test.create({ name: 'test1' }).then(doc => {
    console.log(doc.createdBy); // session user id
    console.log(doc.updatedBy); // session user id
    console.log(doc.createdAt); // created datetime
    console.log(doc.updatedAt); // updated datetime; same as createdAt

    doc.name = 'test2';
    doc.save().then(doc => {
      console.log(doc.updatedBy); // session user id
      console.log(doc.updatedAt); // updated datetime; different from createdAt

      res.send('done');
    });
  });
});
```

The audit option may vary depends on schemas.

```js
const mongAudit = require('express-mong/plugins/audit');
...

const testSchema = new Schema({ name: String, reqPath: String });
const auditOptions = {
  userSchema = 'User', // default to 'User'
  createdBy = 'createdBy', // default to 'createdBy' and false to disable the field
  updatedBy = 'updatedBy', // default to 'updatedBy' and false to disable the field
  createdAt = 'createdAt', // default to 'createdAt' and false to disable the field
  updatedAt = 'updatedAt', // default to 'updatedAt' and false to disable the field
};

testSchema.plugin(mongAudit, auditOptions);
mongoose.model('Test', testSchema);

```
Since audit fields are generated by the plugin, you can omit them in the schema.

### [MIT Licensed](LICENSE)