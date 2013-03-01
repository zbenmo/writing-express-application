Writing Express Application
===========================

Introduction
------------

### Many Bothans died to bring us this information. (I'm following Alex Young [Daily JS](http://dailyjs.com/) who used this phrase in a a blog, and I find it funny).

The "hello world" application with Node/Express is trivial.
Here it is (stolen from [howtonode](http://howtonode.org/getting-started-with-express)):

    var express = require('express'),
        app = express.createServer();

    app.use(express.logger());
  
    app.get('/', function(req, res){
      res.send('Hello World');
    });
  
    app.listen();
    console.log('Express server started on port %s', app.address().port);

The first two lines require *express* and creates the HTTP server (application).
Then we're using a middleware (logger in this case).
And we get to the first and only route we serve */*.
The signature for a route handler (or also for a middleware), is as follows:

    /***
    * @param req - object containing request parameters and intermediate results
    * @param res - object with methods to reply (send, redirect, etc.)
    * @param next - callback function - function(err)
    **/
    function(req, res, next) {
    ..
    }

Finally we listen on the default port (8000), and log to the console.

If you're puzzled how we reach the last line (while listening to the port), this is a basic concept with *node.js*,
everything should be asynchronous and none-blocking. This is where callback functions such as *next* is used.

The challenge is to develop a methodology for building a full fledged web application (talking to databases, serving RESTful APIs as well as web pages & XHR, etc.)

Middleware Revised & Modules
----------------------------

Middleware should have the exact signature as above, and are used either in routes or in *app.use*.
Routes can list more than one middleware. Few examples:

    app.use(express.bodyParser());
    app.use(express.cookieParser());
    
    app.get('/users', loadUsersList, function(req, res) {
       res.render('/users/list.jade',{ req: req });
    });
    app.get('/users:id', loadUser, replyWithUserDetails);

The two routes above "share" the behavior of parsing the body of the request and the cookie.

There is an assumption that both *express.bodyParser* & *express.cookieParser* will each attach their processed output to *req*, and call *next()*.

If indeed the above assumption holds, then we wind up, for example, in */users*. *loadUsersList* not shown here loads the list attach it to *req* and then calls *next()*.

And we render an HTML page with the Jade template engine in this example.

The '/users' route could have been written also as follows:

    app.get('/users', loadUsersList);
    app.get('/users', function(req, res) {
       res.render('/users/list.jade',{ req: req });
    });

What if there was an error on the way? If a middleware (or any route function), calls *next(err)*, then we skip the next middleware functions and look for a function that can handle errors:

    /***
    * @param err - error
    * @param req - object containing request parameters and intermediate results
    * @param res - object with methods to reply (send, redirect, etc.)
    * @param next - callback function - function(err)
    **/
    function(err, req, res, next) {
    ..
    }

For example:

    app.use(express.errorHandler({ dumpExceptions: true }));

What if we discover we can serve the request and nothing else is needed? Just reply and avoid calling *next()*.

    function replyIfUser(req, res, next) {
        if (req.user) {
            res.render('/users/index.jade', { req: req });
            return;
        }
        next(); /// call next if we still don't have a user.
    }

Don't take above example as a recommendation. I'm just showing you what people (and I) may end up doing.

A good practice, in order to avoid calling *next()/next(err)* more than once is to always *return next()* when relevant.

    function ensureUserLoaded(req, res, next) {
        if (req.user) {
            return next();
        }
        User.load(req.id,function(err,user) {
            if (err) {
                return next(err);
            }
            if (!user) {
                return next('not found');
            }
            req.user = user;
            next();
        }
        /// Make sure you understand why the call to next() is placed where it is and not here.
    }

And what if we forget to call *next()*? Like my driving teacher told me, you must make progress. Nothing will happen, the next function in the line will not be called and the end user will don't get the web page.
If you're lucky enough you've used the connection timeout middleware that warns you about it.

    app.use(connectTimeout({
        time : 10000
    }));

