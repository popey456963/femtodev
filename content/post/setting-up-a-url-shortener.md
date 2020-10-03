+++
layout = "post"
title = "Setting Up A URL Shortener"
date = "2016-02-27"
categories = ["Node.JS", "JavaScript", "Project"]
+++

I admit it, yesterday, I built yet another URL shortener with some friends of mine from [Subject Refresh](http://subjectrefresh.info).  Despite most URL shorteners being detrimental to society, with them disappearing and [taking massive sections of the internet](http://archiveteam.org/index.php?title=URLTeam#Dead_or_Broken) with them, we thought we'd add [yet another one](http://subr.pw) to see how well our team worked together.  

We managed to buy the domain, setup the server and host our website in *under an hour*.  This shocked us as we had never done anything in such a short timeframe before.  We seperated jobs as follows:

|Team Member|Main Job|
|---|---|
|[Miles Budden](http://milesbudden.com)|Setting up the domain name.|
|[Finnian Anderson](http://finnian.io/)|Setting up the server and connecting the back and front end.|
|[Alexander Craggs](http://popey456963.github.io)|Building the URL shortener backend.|

Unfortunately, half of our team wasn't around, so we had to all pile in with jobs that we'd never done before.  It meant we learnt a lot about what other members of our team did and was a great experience.

#### **Domain Setup**

The first decision when getting a domain is, of course, choosing the name.  There are [so](http://www.panabee.com/) [many](http://www.leandomainsearch.com/) [domain](http://www.nameboy.com/) [name](http://www.bustaname.com/word_maker) [choosers](http://www.domainhole.com/generator/) out there that it can be intimidating to get help, but I have to say I recommend using [Name Mesh](http://www.namemesh.com/), it has a nice design and assuming you already have a brand name, does a really nice job.  In the end, it gave us the domain name of "[Subr.pw](http://subr.pw)", which we liked a lot, mainly because:

 - It was short, with only six characters.  As we were going to use this for a URL shortening service, it had to be small.
 - It didnt cost very much.  It was 60p for an entire year, which didn't put a dent in our tiny pile of funds.
 - It was related to Subject Refresh and could be seen to have similarities.  You don't want to dillute your brand with severely different domain names.

We actually bought our domain name from [Namecheap](http://namecheap.com), mostly because their customer service is incredible.  They always respond to our tickets within hours and when compared to companies like [GoDaddy](http://godaddy.com), they're better in just about every way.  On that note, we recommend you don't go with GoDaddy, although their prices are great, if you have *any* problem with their service you are not going to get any help from them.  We had tickets open for weeks with no reply, which was why we swapped to a better domain name provider.

#### **Server Configuration**

We already have a server that we use for a lot of [Subject Refresh's](http://subjectrefresh.info) projects.  As we're on rather a tight budget, we went with [Amazon Web Services](https://aws.amazon.com/websites/) as they provide an unbeatable price for the services they provide.  They also provide a surprisingly useful trial service, which allows you to get to grips with their tools.

#### **The Back End**

Our backend is built on MongoDB, Node.JS and Socket.IO to communicate with the frontend.  We used the tremendous [`short`](https://www.npmjs.com/package/short) NPM package to handle the creation and retrieval of the shortened URLs.  Although it's basically a glorified hash store package, it served our purpose well and it has a great syntax that makes it really easy to use.  We originally planned to use [`URL-Shorten`](https://www.npmjs.com/package/url-shorten), however that turned out to have too many pre-requisites and would increase our server load substantially.

The general premise of the [`short`](https://www.npmjs.com/package/short) is that you generate a short URL, you wait until it's finished generating and then you retrieve the shortened URL from the object it returns.  The simplest example is as follows:

```javascript
// Require the Library
var shortURLPromise, short = require('short')

// Connect to MongoDB
short.connect('mongodb://localhost/shortener')

// Generate a Shortened URL
var shortURLPromise = short.generate({
  URL : "http://nodejs.org/"
})

// On Generation, Get the Shortened URL
shortURLPromise.then(function(mongodbDoc) {
	console.log(mongodbDoc)
})
```

Which will return something that looks like:

```yaml
{ URL: 'http://nodejs.org/',
  data: null,
  hash: '292c7c',
  _id: 56d1a8cd1298b7842cb1ff1e,
  created_at: Sat Feb 27 2016 13:46:53 GMT+0000 (GMT Standard Time),
  hits: 0 }
```

This is look very hopeful, it gives us a URL, a hash for the shortened version, a date that it was created on and the number of times it has been requested.  However, we don't want to just access it from the command line, so we're going to have to put some wrapping code around that to communicate between the frontend and the backend.  Enter Socket.IO and Express.

```javascript
// Require Express and Socket.IO
var express = require('express');
var app = express();
var http = require('http').Server(app);
var io = require('socket.io')(http);

// Set the baseURL of the Project
var baseURL = "http://subr.pw/";
// A Regular Expression to Catch URLs
var urlRegExp = /^((ht|f)tps?:\/\/|)[a-z0-9-\.]+\.[a-z]{2,4}\/?([^\s<>\#%"\,\{\}\\|\\\^\[\]`]+)?$/;

// A Wrapper Function for Retrieval from Short
function lookup(long, callback) {
  short.retrieve(long).then(function(result) {
    callback(result.URL);
  });
}

// Set "/static" to be Public
app.use("/", express.static(__dirname + "/static"));

// Setup Redirection from Short URLs
app.get('/:short', function(req, res) {
  lookup(req.params.short, function(redirect) {
    console.log('Long: ' + redirect);
    res.redirect('http://' + redirect);
  });
});

// Homepage of the Website
app.get('/', function(req, res) {
  res.sendFile(__dirname + '/index.html');
});

// Socket.IO Connection
io.on('connection', function(socket) {
  socket.on('url', function(url) {
    url = url.replace("http://", "").replace("https://", "");

    if (url.indexOf("subr.pw") != -1 || !url || !urlRegExp.test(url)) {
      socket.emit("e", {
        message: "Invalid URL"
      });
    } else {
      var shortURLPromise = short.generate({
        URL: url
      });

      shortURLPromise.then(function(mongo_doc) {
        console.log("[Shortener] Converted " + mongo_doc.URL + " to " + baseURL + mongo_doc.hash);
        socket.emit('short', baseURL + mongo_doc.hash);
      });
    }
  });
});

// Listen on Port 3004
http.listen(3004, function() {
  console.log('[Shortener] listening on *:3004')
});
```

And thus, with the communication and URL shortening side done, we need to move onto prettier things.

#### **Front End**

Our front-end was our most contentious point.  None of our team are particuarly good at it, we mostly use frameworks like Materialize or Skeleton.  However, this doesn't mean we can't argue about which one of these to use.  We had the following options:

 - *[Materialize](http://materializecss.com/)* is a versatile Material Design framework that has a sleek layout and works on almost any website.  Unfortunately though, it has rather a large footprint as it supports almost every HTML tag there is.
 - *[Skeleton](http://getskeleton.com/)* is a tiny responsive boilerplate library for making very simplistic/minimalistic sites.  It is very tiny (only ~400 lines), however it is a bit bland for what we wanted to do with it.
 - *Custom Design* was the only way for us to go.  For such a simple website, we figured it couldn't be that hard.  [Codepen.io](http://codepen.io) is a great source of such designs, provided you keep it Open Source and attribute the owner.

 So, we looked around, and we saw a nice simple design.  It wasn't anything special, but we wanted to get this project working in as short a time as possible.  

 <p markdown="0" data-height="236" data-theme-id="0" data-slug-hash="xwDzB" data-default-tab="result" data-user="fluxus" class='codepen'>See the Pen <a href='http://codepen.io/fluxus/pen/xwDzB/'>Simple focus in/out input animation</a> by Mirko Zorić (<a href='http://codepen.io/fluxus'>@fluxus</a>) on <a href='http://codepen.io'>CodePen</a>.</p>
<script async src="//assets.codepen.io/assets/embed/ei.js"></script>

Implementation of Socket.IO into this design was nice and simple.  We simply hooked it up so that when the submit button was pressed, we sent off a `socket.emit` to our server.  We also made it so that when the client received data, it displayed it to them.  

Yet another neat little feature we added was the ability to see the number of URLs created in the time that the user had been there.  

```javascript
$(function() {
  var count = 0;
  var socket = io();
  var url;
  $("form").on("submit", function() {
    url = $("#url").val();
    generateShortlink(url);
    return false;
  });
  $("#url").on("paste", function() {
    var element = this;
    setTimeout(function() { // otherwise the value is blank
      generateShortlink($(element).val());
    }, 0);
  });
  socket.on("increment count", function() {
    count++;
    $("#count").html(count);
  });

  function generateShortlink(url) {
    socket.emit("url", url, function(packet) {
      if (packet.status) {
        $("#count").html(count);
        $("#url").fadeOut("fast", function() {
          $("#url").val(packet.shortlink);
          $("#url").fadeIn("fast");
        });
      } else {
        $("form").effect("shake");
      }
    });
  }
});
```

#### **Continuation**

Evidently, there is much to improve on a project we made in a single hour.  It's been two days, and so far we've improved:

 - Error Handling and URL Validation
 - Creating our own Shortening Engine
 - Statistics on Redirects

You can see all the code [here](http://github.com/subjectrefresh/shortener) and the actual site [here](http://subr.pw).