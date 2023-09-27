---
layout: post
title:  "Using ES6 native promises with the official Node MongoDB driver"
date:   2016-05-25 10:44:47 +0100
categories: node mongodb promises
published: false
---

Lets start by making a simple server with express that has a single POST route which takes in a json string representing a book, parses it with the bodyParser middleware and returns it:

{% highlight javascript %}
'use strict';
const app = require('express')();
const json = require('body-parser').json;

app.post('/api/book', json(), (req, res) => {
  res.status(200).send('okay! you sent: ' + req.body)
});

app.listen(3000, () => console.log('Ready!'));
{% endhighlight %}

Now lets store the json 'book in MongoDB using the official driver. We could try the following:

{% highlight javascript %}
'use strict';
...
const MongoClient = require('mongodb').MongoClient;

app.post('/api/book', json(), (req, res) => {
  MongoClient.connect('mongodb://localhost:27017/booksdb')
    .then((db) => db.collection('books').insertOne(req.body))
    .then((result) => {
      res.status(200).send(result);
    })
    .catch((err) => res.status(503).send('Error: ' + err.message));
});
...
{% endhighlight %}

This works, but the connection made to the database is never closed, naughty naughty. The 'db' object is not accessible in the second 'then', so we are going to need to do some nesting in order to call db.close():

{% highlight javascript %}
app.post('/api/book', json(), (req, res) => {
  MongoClient.connect('mongodb://localhost:27017/booksdb')
    .then((db) => {
      db.collection('books').insertOne(req.body)
        .then((result) => {
          res.status(200).send(result);
          db.close();
        });
    })
    .catch((err) => res.status(503).send('Error: ' + err.message));
});
{% endhighlight %}

That's better, but our code is still nested as it would be if we were using
traditional callbacks. We can improve this by abstracting out the document
insertion:

{% highlight javascript %}
app.post('/api/book', json(), (req, res) => {
  MongoClient.connect('mongodb://localhost:27017/booksdb')
    .then((db) => insertDocument(db, req.body))
    .then((result) => res.status(200).send(result))
    .catch((err) => res.status(503).send('Error: ' + err.message));
});

function insertDocument(db, json) {
  return new Promise((resolve, reject) => {
    db.collection('books').insertOne(json)
      .then((result) => {
        db.close();
        resolve(result);
      })
      .catch((err) => reject(err));
  });
}
{% endhighlight %}
