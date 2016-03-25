passport
==============
This tutorial will show you how to integrate passport authentication into a standard express project.  It is a clone of the project [https://github.com/tutsplus/passport-mongo]

1. run **express passport-test** to create the project
1. To install the packages run **npm install** on the base directory
2. Edit app.js to add the passport and mongo initialization
 * First remove the lines that require the routes
 ```js
var routes = require('./routes/index');
var users = require('./routes/users');
 ```
 * Then remove the lines that use the routes
 ```js
app.use('/', routes);
app.use('/users', users);
 ```
 * And replace them with the mongoose and passport initialization.  Make sure you insert this in place of the "app.use" lines, if you put it above in the "require" section, nothing will work.
 ```js
var mongoose = require('mongoose');
mongoose.connect('mongodb://localhost/passport');
// Configuring Passport
var passport = require('passport');
var expressSession = require('express-session');
app.use(expressSession({
  secret: 'mySecretKey',
  resave: true,
  saveUninitialized: true
}));
app.use(passport.initialize());
app.use(passport.session());

// Initialize Passport
var initPassport = require('./passport/init');
initPassport(passport);

var routes = require('./routes/index')(passport);
app.use('/', routes);
 ```
1. Now create the models directory **mkdir models** and create the file models/user.js with the following
 ```js
var mongoose = require('mongoose');

module.exports = mongoose.model('User',{
    id: String,
    username: String,
    password: String,
    email: String,
    firstName: String,
    lastName: String
});
 ```
1. Now create the passport directory **mkdir passport** and create the file passport/init.js with the following
 ```js
var login = require('./login');
var signup = require('./signup');
var User = require('../models/user');

module.exports = function(passport){

    // Passport needs to be able to serialize and deserialize users to support persistent login sessions
    passport.serializeUser(function(user, done) {
        console.log('serializing user: ');console.log(user);
        done(null, user._id);
    });

    passport.deserializeUser(function(id, done) {
        User.findById(id, function(err, user) {
            console.log('deserializing user:',user);
            done(err, user);
        });
    });

    // Setting up Passport Strategies for Login and SignUp/Registration
    login(passport);
    signup(passport);
}
 ```
1. and passport/login.js
 ```js
var LocalStrategy   = require('passport-local').Strategy;
var User = require('../models/user');
var bCrypt = require('bcrypt-nodejs');

module.exports = function(passport){

    passport.use('login', new LocalStrategy({
            passReqToCallback : true
        },
        function(req, username, password, done) {
            // check in mongo if a user with username exists or not
            User.findOne({ 'username' :  username },
                function(err, user) {
                    // In case of any error, return using the done method
                    if (err)
                        return done(err);
                    // Username does not exist, log the error and redirect back
                    if (!user){
                        console.log('User Not Found with username '+username);
                        return done(null, false, req.flash('message', 'User Not found.'));
                    }
                    // User exists but wrong password, log the error 
                    if (!isValidPassword(user, password)){
                        console.log('Invalid Password');
                        return done(null, false, req.flash('message', 'Invalid Password')); // redirect back to login page
                    }
                    // User and password both match, return user from done method
                    // which will be treated like success
                    return done(null, user);
                }
            );

        })
    );


    var isValidPassword = function(user, password){
        return bCrypt.compareSync(password, user.password);
    }

}
 ```
1. and passport/signup.js
 ```js
var LocalStrategy   = require('passport-local').Strategy;
var User = require('../models/user');
var bCrypt = require('bcrypt-nodejs');

module.exports = function(passport){

    passport.use('signup', new LocalStrategy({
            passReqToCallback : true // allows us to pass back the entire request to the callback
        },
        function(req, username, password, done) {

            findOrCreateUser = function(){
                // find a user in Mongo with provided username
                User.findOne({ 'username' :  username }, function(err, user) {
                    // In case of any error, return using the done method
                    if (err){
                        console.log('Error in SignUp: '+err);
                        return done(err);
                    }
                    // already exists
                    if (user) {
                        console.log('User already exists with username: '+username);
                        return done(null, false, req.flash('message','User Already Exists'));
                    } else {
                        // if there is no user with that email
                        // create the user
                        var newUser = new User();

                        // set the user's local credentials
                        newUser.username = username;
                        newUser.password = createHash(password);
                        newUser.email = req.param('email');
                        newUser.firstName = req.param('firstName');
                        newUser.lastName = req.param('lastName');

                        // save the user
                        newUser.save(function(err) {
                            if (err){
                                console.log('Error in Saving user: '+err);
                                throw err;
                            }
                            console.log('User Registration succesful');
                            return done(null, newUser);
                        });
                    }
                });
            };
            // Delay the execution of findOrCreateUser and execute the method
            // in the next tick of the event loop
            process.nextTick(findOrCreateUser);
       })
    );

    // Generates hash using bCrypt
    var createHash = function(password){
        return bCrypt.hashSync(password, bCrypt.genSaltSync(10), null);
    }
}
     
 ```
1. Now add the routes to routes/index.js
 ```js
var express = require('express');
var router = express.Router();

var isLoggedin = function (req, res, next) {
    if (req.isAuthenticated())
        return next();
    res.sendfile('views/login.html');
}

module.exports = function(passport){

    /* GET login page. */
    router.get('/', isLoggedin, function(req, res){
        res.sendfile('views/info.html');
    });

    /* Handle Login POST */
    router.post('/login', passport.authenticate('login', {
        successRedirect: '/',
        failureRedirect: '/'
    }));

    /* GET Registration Page */
    router.get('/signup', function(req, res){
        res.sendfile('views/signup.html');
    });

    /* Handle Registration POST */
    router.post('/signup', passport.authenticate('signup', {
        successRedirect: '/',
        failureRedirect: '/signup'
    }));

    /* Handle Logout */
    router.get('/signout', function(req, res) {
        req.logout();
        res.redirect('/');
    });

    return router;
}
 ```
1. Now install the packages you will need

 ```
npm install passport --save
npm install passport-local --save
npm install express-session --save
npm install bcrypt-nodejs --save
npm install mongodb --save
npm install mongoose --save
 ```
1. Now create the content.  We will put the files in views, so nobody will be able to get to them through the public directory unless the route sends the file.  Create views/login.html

 ```html
<!DOCTYPE html>
<html>
<head>
<title>Login Page</title>
</head>
<body>
<h1>Sign in to our Passport app
<form action="/login" method="POST">
<input type="text" name="username" placeholder="Email" required autofocus>
<input type="password" name="password" placeholder="Password" required>
<button type="submit">Sign in</button>
</form>
<a href="/signup">Create an account</a>
</body>
</html>
 ```
1. and views/signup.html

 ```html
 <!DOCTYPE html>
<html>
<head>
<title>Signup</title>
</head>
<body>
<h1>Registration Details<div>
<form action="/signup" method="POST" >
<input type="text" name="username" placeholder="Username" required autofocus >
<input type="password" name="password" placeholder="Password" required >
<input type="email" name="email" placeholder="Email" required>
<input type="text" name="firstName" placeholder="First Name" required>
<input type="text" name="lastName" placeholder="Last Name" required>
<button type="submit">Register</button>
</body>
</html>
 ```
1. and views/info.html

 ```html
 <!DOCTYPE html>
<html>
<head>

<title>Login Page</title>
</head>
<body>
<h1>This is the info page</h1>
If you want to logout, go <a href="/signout">here</a>.
</body>
</html>
 ```
1. And now run it with **npm start**
