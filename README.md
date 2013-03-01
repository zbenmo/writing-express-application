Writing Express Application
===========================

Introduction
------------

### Many Bothans died to bring us this information.
(I'm following Alex Young [Daily JS](http://dailyjs.com/) that used this phrase in a a blog, and I find it funny).

The "hello world" application with Node/Express is trivial.
Here it is (stolen from [howtonode](http://howtonode.org/getting-started-with-express)):

>var express = require('express'),
>    app = express.createServer();
>
>app.use(express.logger());
>
>app.get('/', function(req, res){
>    res.send('Hello World');
>});
>
>app.listen();
>console.log('Express server started on port %s', app.address().port);

The first two lines require *express* and creates the HTTP server (application).
Then we're using a middleware (logger in this case).
And we get to the first and only route we serve */*.
The signature for a route handler (or also for a middleware), is as follows:

>/***
>* @param req - object containing request parameters and intermediate results
>* @param res - object with methods to reply (send, redirect, etc.)
>* @param next - callback function - function(err)
>**/
>function(req, res, next) {
>..
>}

Finally we listen on the default port (8000), and log to the console.
If you're puzzled how we reach the last line (while listening to the port), this is a basic concept with *node.js*,
everything should be asynchronous and none-blocking. This is where callback functions such as *next* is used.
