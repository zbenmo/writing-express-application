Writing Express Application
===========================

Introduction
------------

### Many Bothans died to bring us this information. (I'm following Alex Young [Daily JS](http://dailyjs.com/) who used this phrase in a blog, and I find it funny).

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

### Middleware (Revised) & Modules

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

If indeed the above assumption holds, then we wind up, for example, in */users*. *loadUsersList* not shown here loads the list, attaches it to *req* and then calls *next()*.

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

A good practice, in order to avoid calling *next()/next(err)* more than once by mistake, is to always *return next()* when relevant.

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

And what if we forget to call *next()*? Like my driving teacher told me, you must make progress. Nothing will happen, the next function in the line will not be called and the end user will not get the web page.
If you're lucky enough you've used the connection timeout middleware that warns you about it.

    app.use(connectTimeout({
        time : 10000
    }));

### Modules

To break a monelitic *app.js* file into manageble project, we can use the following structure (I justed used the scafold express generated application for this example).

    .
    ├── app.js
    ├── package.json
    ├── public
    │   ├── images
    │   ├── javascripts
    │   └── stylesheets
    │       └── style.css
    ├── routes
    │   ├── index.js
    │   └── user.js
    └── views
        ├── index.jade
        └── layout.jade

Where, for example, *user.js* is:

    /*
     * GET users listing.
     */

    exports.list = function(req, res){
      res.send("respond with a resource");
    };

And *app.js* is:

    var express = require('express')
      , routes = require('./routes')
      , user = require('./routes/user')
    
    ...
    
    app.get('/users', user.list);
    
    ...

### What if we want to call one or more middleware from within another middleware?

    function ensureUserLoaded(req, res, next) {
        if (req.user) {
            return next();
        }
        loadUser(req, req, next);
    }

Or if we want to add some logic:

    function ensureUserLoaded(req, res, next) {
        if (req.user) {
            req.user.smile = true;
            return next();
        }
        loadUser(req, req, function(err) {
            if (err) {
                return next(err);
            }
            if (!req.user) {
                req.user = new User({smile: true});
            } else {
                req.user.smile = true;
            }
            next();
        }
    }

Or maybe we need two functions to be called one after the other:

    function ensureUserLoaded(req, res, next) {
        if (req.user) {
            return next();
        }
        loadUser(req, req, function(err) {
            if (err) {
                return next(err);
            }
            if (!req.user) {
                req.user = new User();
            }
            next();
        }
    }

    function makeUserSmile(req, res, next) {
        assert(req.user);
        req.user.smile = true;
        next();
    }

    app.get('/', ensureUserLoaded, makeUserSmile, function(req, res) {
        res.render('users/smiling.jade', { req: req });    
    });

Code and functionality (middleware) can be shared beteen routes, by *using* it, or explicitly when only some of the routes need it, or from within other middleware.

    app.use(loadUser);
    
    app.get('/', renderTop);
    app.get('/users/:id', renderProfile);
    
    Or
    
    app.get('/', loadUser, renderTop);
    app.get('/users/:id', loadUser, renderProfile);
    app.get('/users', loadList, renderList); /// here we don't need loadUser
    
    Or
    
    app.get('/', top);
    app.get('/users:id', profile);
    
    function top (req, res, next) {
        loadUser(req, res, function(err) {
            if (err) return next(err);
            res.render('/top', {req: req}); /// and I'll not call next()
        });
    }
    
    function profile  (req, res, next) {
        loadUser(req, res, function(err) {
            if (err) return next(err);
            res.render('/profile', {req: req}); /// and I'll not call next()
        });
    }

Small hint. If you wonder what gets run first, the middleware, or the routes, and if the order is important, then yes, the order is important, and you can also search for Express documentation about **app.route**.
This, is also relevant for the error handling.

You may feel by now the potential but also the pain. There are a lot of opportunities to reuse code and share between projects, and yet it can be confusing.

### Should all our functions from now and forever have the middleware signature, *function(req, res, next)* ?

I feel a little wrong about misusing *req* for everything, and about passing callbacks even for clearly local synchronous functionality.
There are also questions of how the testing should look like.
If nothing better, then we can use asserts and documenation.

    /***
    * expects req.task
    * If successful will set req.result
    * Will not reply to end user (will call next()/next(err))
    **/
    function underTest (req, res, next) {
        assert(req.task);
        
        ...
        next();
    }
    
    --

    ...
    it('should perform the task', function(done) {
        
        var req = {task : {add:{l: 1, r: 2}}};
        var res = null;
        
        underTest(req, res, function(err) {
            assert(!err);
            assert(req.result === 3);
            done();
        }
        
    });

Also, if we want to leave the *app.js* as light as possible, and put the logic in the "routes" themselves, it is less clear how to arrange the common logic.

    app.js
    ------
    
    app.get('/', routes.top);
    app.get('/users', users.list);
    
    routes.js
    ---------
    
    /***
    * Ahm, will render if everything okay, but if an error occured will call next(err)
    */
    export.top = function(req, res, next) {
        /// request already parsed?
        /// have we already loaded a user?
        /// Am I alowed to reply from here, and avoid calling next() ?
        ...
        assert(req.params);
        assert(!req.user);
        ...
    }
    
    users.js
    --------
    
    /***
    * Will not reply to end user (will call next()/next(err))
    */
    export.list = function(req, res, next) {
        /// request already parsed?
        /// have we already loaded a user list?
        /// Am I alowed to reply from here, and avoid calling next() ?
        ...
        assert(!req.params);
        assert(req.usersList);
        ...
    }

### Reminder of How We Got Here (related to Egypt & Mexico)

Asynchronous code with callback, will put us in a situations where we build pyramids.

    func_a(1,3,function(err,res) {
        if (err) {
            err_handling1(err,function() {
                return -1;
            }
        } else if (res) {
            func_b(res,42,function(err1,res1) {
                if (err1) {
                    return -2;
                }
                return res1;
            }
        } else {
            return -3;
        }
    });

(and I have been gentle)

Here is another example, where we use functions with the middleware signature.

    function a (req, res, next) {
        if (req.user) {
            b(req, res, function(err) {
                if (err && (err.code === 4)) {
                    return a_1(req, res, next); /// we use the the original callback
                } else if (err) {
                    return next(err);
                } else if (req.hobby) {
                    return c(req, res, next); /// same here
                }
                return next();
            });
        }
        b_1(req, res, function(err) {
            assert(!err);
            c_1(req, res, function(err) {
                if (err) console.error(err.message);
                next(err); /// err is passed wheather defined or not
            });
        });
    }

I admit I added the comment to make the pyramid a little taller, yet you got the point.

### Async

[Async.js](https://github.com/caolan/async) (or another similar utility), can be part of the solution. It enables handling asynchronized code a little more systematicly. Taken from there:

    async.series([
        function(callback){
            // do some stuff ...
            callback(null, 'one');
        },
        function(callback){
            // do some more stuff ...
            callback(null, 'two');
        }
    ],
    // optional callback
    function(err, results){
        // results is now equal to ['one', 'two']
    });

Note that in all above examples, I've avoided loops, and also avoided parallelism in the calls to the building blocks, as those add yet more complexity. *Async.js* and the alternatives, can help significantly also with loops and parallelism.

Suggested Methodology
---------------------


    
    
