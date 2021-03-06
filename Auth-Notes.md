# Authentication
[toc]
## Introduction
1. These notes follow the major portion of Stephen Grider's course on Udemy, *Advanced React and Redux*, although a large portion of this material will have nothing to do with React/Redux, but with the back end tasks of authoriztion. It can be seen as a summary of a many topics all bundled together, so the following notes might seem a bit disjointed.

2. Be sure to see the completed app in the *apps* subdirectory to see the details of the items discussed herein.

3. Architecture will be a major part of this project. There will be a separate server to handle authentication, and this will have nothing to do with React (or Redux). All that the React front-end requires is that it get served valid JSON data, and it doen't matter what kind of server does it, *apache*, *nginx*, *node*, *etc.*

## Authentication Generally
1. For a very helpful overview, look in the *Diagrams* directory for the file *Authentication Flow.png*, which shows from a high level the steps involved and where the steps are performed.

2. One key concept is that we only want to authenticate a user one time, but we need to know every time the user makes a request to the server for a protected resource, are they authorized or not. This is done by having a single, up-front, request for username/password to authenticate, then passing to the user an identifying token to last the session. The token will get added to each request behind the scenes, so the user will not have to deal with it. When the session ends, the token should disappear.

### Cookies vs. Tokens
1. Once the user is authenticated (*i.e.*, he has entered his username and password, and been recognized as a valid user), he will receive a piece of information that identifies him as authorized. This may be in the form of a **cookie** or a **token**, each of which is described below.

2. A **cookie** is a chunk of JSON data that is included automatically in the header of each and every request sent to the domain for which that cookie is associated. It cannot be sent to other domains. At root, they are an attempt to bring state into an inherently stateless environment.

3. A **token** is simply a piece of data, usually a long identifying string. They are **not** included automatically in the request, and must be wired up manually; *i.e.*, we will add a header property such as:
    ```
    authorization: asdf9wer3ww
    ```
4. One important thing about tokens is that they **can** be sent to any domain. So, if our application is distributed over serveral servers on different domains, we can use the single token to access any of them, whereas we would need a separate cookie for each separate domain.

5. Generally, the world is heading towards tokens as the session verification solution, and that is what we will use in these notes.

## Backend
### Basic Setup
1. See the *Server* app in this directory for any details; however, the backend begins with a standard Node/Express/Mongo setup, to save users who submit an e-mail and a password.

### Password Encryption
1. So far, we have set up a system where our back end saves a new user's email and password; however, it is a **major malpractice**, from a security perspective, to save the passwords to the database in an unencrypted format.

2. We must always encrypt passwords prior to saving. In the example herein, we will use the **bcrypt** library for encryption.

3. The first step is to import the library using npm:
    ```
    npm install --save bcrypt-nodejs
    ```
4. In the *users.js* user model file, we need to require bcrypt and add the following code:
    ```javascript
    // on save, encrypt password
    UserSchema.pre('save', function(next) {
        const user = this;

        bcrypt.genSalt(10, (err, salt) => {
            if (err) {
                return next(err);
            }
            bcrypt.hash(user.password, salt, null, (err, hash) => {
                if (err) {
                    return next(err);
                }
                user.password = hash;
                next();
            });
        });
    });
    ```
    **pre**: this is a mongoose model hook that provides functionality to be run either before (*pre*) or after (*post*) the event passed in as the first parameter. 
    
    **this**: Note that the value of **this**, which is assigned to "user" in the the above code is the user that is about to be saved. Therefore, it is important that the first callback be in traditional form, so that its *this* context is the user from the userSchema object, and not the outside scope if an arrow function is used. However, if arrow functions are used on the inner callbacks, we can access *this* directly, without needing the assignment to *user*.
    
    **bcrypt**: For more information, see the documentation for bcrypt; however, at this stage it is making a "salt", which is then combined with the hashed (*i.e.*, encrypted) password to make the new password.
    

### Assigning a Token
1. Once a user has registered with an email and a password, and the password has been encrypted and stored in our database, we need to assign the user a **token**, which will be a piece of identifying information which the user will present with every future request in their session.

2. What we will give them is called a **JWT** or **JSON Web Token**. As seen below, it will be created by a combination of the user ID and a secret string. When submitted along with the request, the *JWT* will be "unlocked" to give us the user ID.

3. Note that the JWT is an open standard for securely transmitting information. So, the object that we have upon decryption can contain a wide variety of information. There is a list of *reserved* properties, such as *iat* for "issued at". 

3. It should be pretty obvious that it is imperative that the "secret string" must be absolutely kept secret. Do not post on gitHub,*etc.*

4. To generate the *JWT*, we will install a library called **jwt-simple** with npm.

5. In the root directly, we can set up a *config.js* file to hold our secret string. Make sure this file is added to the *.gitignore* file. 

6. Next, to work a little bit backwards, we can change our return after saving a new user to something like:
    ```javascript
    //authController.js
    newUser.save()
    .then(() => {
        // respond to request indicating the user was created
        res.status(200)
            .json({ token: tokenForUser(newUser) });
    })
    ```
7. Finally, we should require in the *jwt* library, and create a method to create our token:
    ```javascript
    function tokenForUser(user) {
        const timestamp = new Date().getTime();
        return jwt.encode({ sub: user.id, iat: timestamp }, config.secret);
    }
    ```

### Policing Route Access with Passport
1. At this point, we are signing up users, and giving them a token upon signing up. However, we still aren't doing anything for users that are already registered and are logging in; also, we need to have a means of verifying a user's token before allowing access to a protected resource (see the flow chart in the diagrams).

2. Note that we want to check status only on routes with protected resources, while there may be many routes that should be accessible by anyone.

3. We will handle this problem with a library called **passport**. Although it is usually used for cookie-based authentication, we will be able to use it with our JWTs. We start with the following installation:
    ```
    npm install --save passport passport-jwt
    ```
    
4. Passport can best be thought of as an ecosystem, allowing access to a variety of different authentication **strategies**. In our app, for example, we could be using a JWT verification strategy and a username/password strategy.

5. Next, we should set up a "services" directory on the project root, and place a file, **passport.js** inside that folder. This file will be used to set up a strategy and then, at the end, we tell passport to apply our strategy as middleware with the line:
    ```javascript
    passport.use([strategyName]);
    ```
6. Then, in our routes page (*router.js*), we **require in** our passport services file (we don't have to assign it to a variable or call it anywhere - simply requiring it will cause it to run, including the last line presented above). Then we add the strategy to each proptected route as follows:
    ```javascript
    const Authentication = require('./controllers/authController');
    require('./services/passport');
    const passport = require('passport');

    const requireAuth = passport.authenticate('jwt', { session: false });

    module.exports = function(app) {
        app.get('/', requireAuth, function(req, res) {
            . . .
        });

        app.post('/signup', Authentication.signup);
    }
    ```
    **requireAuth** is a key line in the above code, where we assign to the variable the passport strategy, which is idendtified as a web token strategy by the first parameter. The second parameter overrides the default passport setting of making a cookie-based session upon successful auth. We do not want that, as we are using the token-based authentication.

7. Now, back to the *services/passport.js* file, where we set up our auth strategy. There is a bit of behind-the-screen magic going on here, so not everything is traceable on its face, but the comments provided should give pretty good direction:
    ```javascript
    const passport = require('passport');
    const User = require('../models/users');
    const config = require('../config');
    const JwtStrategy = require('passport-jwt').Strategy;
    const ExtractJwt = require('passport-jwt').ExtractJwt;

    // set up options for JWT Strategy
    // note that we are adding our token to the header
    // we have to tell passport where to look for it
    const jwtOptions = {
        jwtFromRequest: ExtractJwt.fromHeader('authorization'),
        secretOrKey: config.secret
    };

    // create jwt Strategy
    // the payload is the decoded JWToken
    // done is a callback we need to call depending on success of auth
    const jwtLogin = new JwtStrategy(jwtOptions, function(payload, done) {
        // check if userID in the payload exists in our database
        // if yes, call done() with the user as a parameter
        // if no, call done without the user object
        User.findById(payload.sub, function(err, user) {
            // handle situation such as communication failure
            if (err) {
                return done(err, false);
            }
            // successful login, move on with user
            if (user) {
                done(null, user);
            } else {
                // auth worked, but user was not authorized
                done(null, false);
            }
        });
    });

    // tell passport to use this strategy
    passport.use(jwtLogin);
    ```
### Adding Sign-In Capabilities
1. So far, we have a way for a new user to register and get a token. We also have a way to check the token when a route is accessed and make certain the user is authorized.  Now, we will address the need of a registered user to log in and get a token to allow access.

2. This will involve a new passport strategy, one of checking the username and password against our database, so we will need to set it up. To start off, we need to run at the command line:
    ```
    npm install --save passport-local
    ```
3. In our *services/passport.js* file, we need to set up the **local strategy**, which is how the username/password authorization is termed.  So, we begin by requiring in our *passport-local* library, and inser the following code:
    ```javascript
    const LocalStrategy = require('passport-local');

    const localOptions = { usernameField: 'email'};
    const localLogin = new LocalStrategy(localOptions, function(email, password, done){
	
    });
    ```
    **localOptions**: by default, the local-strategy expects a username as a propery called "username". Since our property is termed "email", we have to override the default value of the *usernameField*.

4. One thing we will need to do is prepare a way to check the entered password against the hashed and salted password contained in the database for the user. To do this, we will add an instance method to the UserSchema object, which will call on the *bcrypt* library:
    ```javascript
    // models/users.js
    userSchema.methods.comparePassword = function(candidatePassword, callback) {
        bcrypt.compare(candidatePassword, this.password, (err, isMatch) => {
            if (err) {
                return callback(err);
            }
            callback(null, isMatch);
        });
    }
    ```
    **Note**: The Schema has a *methods* property which is an object containing any methods assigned to instances of the UserSchema. So, wherever we need to use it, we can import the UserSchema and then use the **comparePassword()** method.

5. So, we will use our *comparePassword* method in our **localLogin()** method:
    ```javascript
    const localLogin = new LocalStrategy(localOptions, function(email, password, done){
        User.findOne({ email: email })
        // here we verify the email and password, then call
        .then((user) => {
            if (!user) {
                return done(null, false)
            }
            //compare passwords
            user.comparePassword(password, (err, isMatch) => {
                if (err) {
                    return done(err);
                }
                // if not, call done with false as the second parameter
                if (!isMatch) {
                    return done(null, false);
                }
                // done if it matches known email and password
                return done(null, user);
            });
        })
	.catch((err) => done(err));
    });
    ```
    Of course, do not forget to include "passport.use(localLogin)" at the end of the *passport.js* file.
    **Important**: The *done()* callback is supplied by *passport*, and one of the things it does is, if there is a *user*, it assigns it to the *req* object.


## Frontend
1. In this section, we are going to create a front-end interface to allow the user to sign-in, sign-up, or sign-out. There will be four pages, from the user's perspective:

    a. a landing page, where the user will first arrive,
    
    b. a sign-in page, where a registered user can sign in,
    
    c. a sign-up page, where a new user can sign up, and
    
    d. a feature page, which will contain a protected route.
    
2. We will not store our token in our application state, but will store it within our browser session.  Our **application state** will manage two items of information:
    ```json
    {
        auth: {
            authenticated: BOOLEAN,
            error: STRING
        }
    }
    ```
3. We will have action creators for signing in, signing out, and for signing up to toggle the authenticated property.

### General Flow
1. After we create a sign-in form, we will need to make an *action creator* to handle submitting the e-mail/password combination to the servrer. It will get back **either** a negative response, or a JWT token. We need to handle each branch, by showing an error message if no login occurs, or by redirecting an authenticated user to the start-up page, saving the token, and updating application state.

2. In order to have greater flexibility in handling the various possible returns from the back end, we will use **redux-thunk** instead of **redux promise** as our middleware.

### CORS
1. One frustrating error that we will receive using our current setup of a front end making AJAX requests to a back-end server is the "No 'Access-Control-Allow-Origin' header is present on the requested resource".

2. **CORS** stands for **cross origin resource sharing**. It is a security concept for the protection of users in a browser environment, to prevent a browser from accessing other websites.

3. Under CORS, the domain and port of the requesting device is attached to every request that is made. If we look at the *request headers*, we will see a *host* and an *origin* property.  The requested server can then inspect this domain and port and, if different from the server, can refuse the request.

4. CORS issues cannot be solved on the client side alone. To fix CORS issues, we can change our server to allow the requests.

5. In order to fix the problem, we change our API to allow such requests. To do so, take the following steps:

    a. In the directory of the back-end server, run the following:
    ```
    npm install --save cors
    ```
    
    b. In the *index.js* file, require in cors, and then add it to the middleware stack:
    ```javascript
    const cors = require('cors');
    
    . . .
    
    app.use(morgan('combined'));
    app.use(cors());
    app.use(bodyParser.json({ type: '*/*' }));
    router(app);
    
    . . .
    ```
### Handling the JWT
1. To save our JWToken, we will use **localStorage**. localStorage is memory dedicated to the browser on the client machine, **not** on the server. It can be thought of as local hard drive access.

2. Localstorage cannot be shared among the user's different devices.

3. By storing in localStorage, the JWT will be easily accessible to be attached to any requests that require authentication.

4. Localstorage outlasts the session, so the next time the user revisits the site, the JWT is still available, unless it has been deleted.

5. **Localstorage is not shared across domains!**

## Protecting Front-End Routes with JWT
1. Once we have created our back-end and our signin/signup pages on the front end, we can now register as a user, and log in as a registered user. On the back end, we will be restricted from accessing routes we are not authorized to access, and this is the more serious goal, because we want to protect our database, etc.  But we will also want to make a good user experience by limiting access to pages not authorized to visit, and providing feedback.

2. What we will do is check our state to see if the authenticated property is true. Then, we can protect those routes we wish by wrapping them with a higher-order component to add the authentication functionality.

3. Our Higher Order component will look as follows (for more discussion on HOCs, see the React Notes):
    ```javascript
    // require_auth.js
    import React, { Component } from 'react';
    import { connect } from 'react-redux';

    export default function (ComposedComponent) {
        class Authentication extends Component {
            componentWillMount() {
                if (!this.props.authenticated) {
                    this.props.history.push('/signup');
                }
            }
            componentWillUpdate(nextProps) {
                if (!nextProps.authenticated) {
                    this.props.history.push('/signup');
                }
            }
            render() {
                return <ComposedComponent {...this.props} />;
            }
	}

	const mapStateToProps = state => (
            {
                authenticated: state.auth.authenticated
            }
        );

        return connect(mapStateToProps)(Authentication);
    }
    ```
4. Then, in our *index.js* file, we can wrap protected routes as follows:
    ```javascript
    import RequireAuth from '[path to HOComponent';
    <Route path="/feature" component={RequireAuth([component]} />
    ```
    
## Making Authenticated API Requests
1. Once the user logs in, we have access to the JWT token in localStorage.
