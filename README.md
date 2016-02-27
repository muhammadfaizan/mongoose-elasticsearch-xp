# mongoose-elasticsearch-xp

[![NPM version][npm-image]][npm-url] [![Build Status][travis-image]][travis-url] [![Coverage Status][coverage-image]][coverage-url]

mongoose-elasticsearch-xp is a [mongoose](http://mongoosejs.com/) plugin that can automatically index your models into [elasticsearch](http://www.elasticsearch.org/).


- [Why this plugin?](#why-this-plugin)
- [Installation](#installation)
- [Setup](#setup)
- [Indexing](#indexing)
  - [Saving a document](#saving-a-document)
  - [Indexing nested models](#indexing-nested-models)
  - [Indexing an existing collection](#indexing-an-existing-collection)
  - [Indexing on demand](#indexing-on-demand)
- [Mapping](#mapping)
  - [Creating mappings on-demand](#creating-mappings-on-demand)

## Why this plugin?

Although mongoosastic is a great tool, it didn't fit my needs. I needed something more flexible and up to date than mongoosastic.  
I started by sending some pull requests to mongoosastic. When I was facing to a full rewrite need, I choosed to start a new project based on the mongoosastic idea / syntax.  
This plugin handle both callback and promise syntaxes. It uses the mongoose Promise which can be [redefined](http://mongoosejs.com/docs/promises.html#plugging-in-your-own-promises-library)

## Installation

The latest version of this package will be as close as possible to the latest `elasticsearch` and `mongoose` packages. 

```bash
npm install --save mongoose-elasticsearch-xp
```

## Setup

### Model.plugin(mongoose-elasticsearch-xp, options)

Options are:

* `index` - the index in Elasticsearch to use. Defaults to the collection name.
* `type`  - the type this model represents in Elasticsearch. Defaults to the model name.
* `client` - an existing Elasticsearch `Client` instance.
* `hosts` - an array hosts Elasticsearch is running on.
* `host` - the host Elasticsearch is running on
* `port` - the port Elasticsearch is running on
* `auth` - the authentication needed to reach Elasticsearch server. In the standard format of 'username:password'
* `protocol` - the protocol the Elasticsearch server uses. Defaults to http


To have a model indexed into Elasticsearch simply add the plugin.

```javascript
var mongoose = require('mongoose'), 
    mexp = require('mongoose-elasticsearch-xp');

var User = new Schema({
    name: String, 
    email: String, 
    city: String
});

User.plugin(mexp);
```

This will by default simply use the collection name as the index while using the model name itself as the type. 
So if you create a new User object and save it, you can see it by navigating to http://localhost:9200/users/user/_search 
(this assumes Elasticsearch is running locally on port 9200). 

The default behavior is all fields get indexed into Elasticsearch. 
This can be a little wasteful especially considering that the document is now just being duplicated between mongodb and Elasticsearch so you should consider opting to index only certain fields by specifying `es_indexed` on the fields you want to store:


```javascript
var User = new Schema({
    name: {type:String, es_indexed:true}, 
    email: String, 
    city: String
});

User.plugin(mexp);
```

In this case only the name field will be indexed for searching.

Now, by adding the plugin, the model will have a new method called `esSearch` which can be used to make simple to complex searches.  
The `esSearch` method accepts [standard Elasticsearch query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)

```javascript
User
  .esSearch({
    query_string: {
      query: "john"
    }
  })
  .then(function(results) {
    // results here
  });
```

The `esSearch` also handle the full Elasticsearch ...

```javascript
User
  .esSearch("name:john")
  .then(function(results) {
    // results here
  });
```

... and Lucene syntax:

```javascript
User
  .esSearch({
    query: {
      match_all: {}
    }, 
    filter: {
      range: {
        age: {lt: 35}
      }
    }
  })
  .then(function(results) {
    // results here
  });
```

To connect to more than one host, you can use an array of hosts. 

```javascript
MyModel.plugin(mexp, {
  hosts: [
    'localhost:9200',
    'anotherhost:9200'
  ]
})
```

Also, you can re-use an existing Elasticsearch `Client` instance

```javascript
var esClient = new elasticsearch.Client({host: 'localhost:9200'});

MyModel.plugin(mexp, {
  client: esClient
});
```


## Indexing

### Saving a document
The indexing takes place after saving inside the mongodb and is a defered process. 
One can check the end of the indexion catching es-indexed event. 

```javascript
doc.save(function (err) {
  if (err) throw err;
  /* Document indexation on going */
  doc.on('es-indexed', function (err, res) {
    if (err) throw err;
    /* Document is indexed */
    });
  });
```


###Indexing Nested Models
In order to index nested models you can refer following example.

```javascript
var Comment = new Schema({
    title: String, 
    body: String, 
    author: String
});

var User = new Schema({
    name: {type:String, es_indexed:true}, 
    email: String, 
    city: String, 
    comments: {type:[Comment], es_indexed:true}
});

User.plugin(mexp);
```

### Indexing An Existing Collection
Already have a mongodb collection that you'd like to index using this plugin? 
No problem! Simply call the `esSynchronise` method on your model to open a mongoose stream and start indexing documents individually.

```javascript
var BookSchema = new Schema({
  title: String
});
BookSchema.plugin(mexp);

var Book = mongoose.model('Book', BookSchema);

Book.on('es-bulk-sent', function () {
  console.log('buffer sent');
});

Book.on('es-bulk-data', function (doc) {
  console.log('Adding ' + doc.title);
});

Book.on('es-bulk-error', function (err) {
  console.error(err);
});

Book
  .esSynchronize()
  .then(function () {
    console.log('end.');
  });
```

`esSynchronise` use same parameters as [find](http://mongoosejs.com/docs/api.html#model_Model.find) method.
It allows to synchronize a subset of documents, modifying the default projection...

```javascript
Book
  .esSynchronize({author: 'Arthur C. Clarke'}, '+resume')
  .then(function () {
    console.log('end.');
  });
```

### Indexing On Demand
You can do on-demand indexes using the `esIndex` function

```javascript
Dude.findOne({name:'Jeffrey Lebowski', function(err, dude) {
  dude.awesome = true;
  dude.esIndex(function (err, res) {
    console.log("egads! I've been indexed!");
  });
});
```

Note that indexing a model does not mean it will be persisted to
mongodb. Use save for that.

## Mapping

### Creating Mappings On Demand

You can do on-demand create a mapping using the `esCreateMapping` function.

Creating the mapping is a one time operation and can be done as follows:

```javascript 
var User = new Schema({
    name: String, 
    email: String, 
    city: String
});

User.plugin(mexp);

User
  .esCreateMapping({
    "analysis" : {
      "analyzer":{
        "content":{
          "type":"custom",
          "tokenizer":"whitespace"
        }
      }
    }
  })
  .then(function (mapping) {
    // do neat things here
  });

```

You'll have to manage whether or not you need to create the mapping, mongoose-elasticsearch-xp will make no assumptions and simply attempt to create the mapping. 
If the mapping already exists, an Exception detailing such will be populated in the `err` argument. 


[npm-url]: https://npmjs.org/package/mongoose-elasticsearch-xp
[npm-image]: https://badge.fury.io/js/mongoose-elasticsearch-xp.svg

[travis-url]: http://travis-ci.org/jbdemonte/mongoose-elasticsearch-xp
[travis-image]: https://secure.travis-ci.org/jbdemonte/mongoose-elasticsearch-xp.png?branch=master

[coverage-url]: https://coveralls.io/github/jbdemonte/mongoose-elasticsearch-xp?branch=master
[coverage-image]: https://coveralls.io/repos/github/jbdemonte/mongoose-elasticsearch-xp/badge.svg?branch=master
