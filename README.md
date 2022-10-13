# Tay Auth Server Middleware

This is an express middleware and router to be used with the
[@taylorgrinn/auth](https://github.com/taylorgrinn/auth.git) react
component.

# Installation

```cmd
npm i --save @taylorgrinn/auth-server
yarn add @taylorgrinn/auth-server
```

# Add the middleware and router to your app

```js
import bodyParser from 'body-parser';
import express from 'express';
import useAuth from '@taylorgrinn/auth-server';

const [authMiddleware, authRouter] = useAuth({
  // Provide a secret key or list of keys to use for creating sessions.
  secret: 'ssshhh',
  // In order to implement the reset password form, use this option to send
  // an email using nodemailer or another email solution. Must return a promise
  async sendCode(email, code) {
    console.log(`Emailing code: ${code} to ${email}`);
    /* Implementation not shown */
  },
  delay: 1000, // Add an optional delay to all authentication requests. Useful for preventing brute force attacks from a single IP or just letting users appreciate your loading screens.
});

const app = express();
export default app;

app.use(
  bodyParser.json(), // JSON necessary to parse requests from the @tygr/auth react component
  authMiddleware
);

app.use('/auth', authRouter);
```

This is all the setup you need for a minimal working example with local login, register, and reset password functionality.

# Setup the external providers (Google, Github, Facebook)

Create API keys for each or any of the services:

- [Google](https://console.developers.google.com/)
- [Github](https://github.com/settings/apps)
- [Facebook](https://developers.facebook.com)

Add the keys, as well as the external location of your authentication server, to the `useAuth` options:

```js
const [authMiddleware, authRouter] = useAuth({
  ...,
  authBaseUrl: 'https://tygr.info/api/auth', // Used for callbacks after authentication
  google: {
    clientId: 'blahblahblah',
    clientSecret: 'blahblahblah',
    scope: [], // Optional: scopes allow you to get more user information
  },
  github: {
    clientId: 'blahblahblah',
    clientSecret: 'blahblahblah',
    scope: [], // Optional: scopes allow you to get more user information
  },
  facebook: {
    clientId: 'blahblahblah',
    clientSecret: 'blahblahblah',
    scope: [], // Optional: scopes allow you to get more user information
    profileFields: [] // Optional: profile fields need to be specified for user object
  }
});

...

/**
 * Make sure to include the pathname you decide to use here ('/auth')
 * in the `authBaseUrl` option above.
 */
app.use('/auth', authRouter)
```

For the `scope` options, the `email` scope is already included in all
authentication requests by default for both github and google.

When creating the keys for each provider, you'll also have to specify
the callback urls in this format:

- Google: `` `${authBaseUrl}/google/callback` ``
- Github: `` `${authBaseUrl}/github/callback` ``
- Facebook: `` `${authBaseUrl}/facebook/callback` ``

# Setup better storage options

By default, the sessions and user data are stored in-memory. This is
volatile and not recommended for production use. You may pass in a
`store` for the sessions and a `Users` model for storing users.

The `store` interface is [described
here](https://github.com/expressjs/session#session-store-implementation)
in the express-session library.

The `Users` interface defines what will probably be static methods on
your `Users`' model class. I designed it specifically to work with
sequelize right out of the gate, but it can be adapted for any storage
mechanism you'd like. There are five methods to implement and four
required fields on the User object.

---

**BaseUser** (You may add any other properties you'd like)
| property | type | description |
| -------- | --------------------- | ---------------------------------------------------------------------------- |
| id | string | Unique identifier generated by uuid. This should be the 'primary key' |
| email | string | Unique email address for each user. |
| password | string (or undefined) | Hashed password for the user if they signed in locally |
| provider | string | The authentication method (either 'local', 'google', or 'github') |

---

**BaseUsers**
| method | param | returns | description |
| -------------- | --------- | --------------------- | ----------------------------------------------------------------------------------------------------------- |
| create | User | Promise\<User> | Create a new user. No need to check if the user already exists. For sequelize, no setup is necessary as the base model includes this method with the correct signature, given that the model extends the `BaseUser` interface correctly |
| findByEmail | string | Promise\<User \| null> | Find a user account by an email address |
| findAndUpdate | User | Promise\<any> | Find a user using the id, which will not change, and update the rest of the user properties |
| findAndDestroy | User | Promise\<any> | Find a user using either the id or email then delete them |
| sanitize | User | Partial\<User> | Synchronously create a user object that is safe to be sent back to the user. The password hash should be removed at least |

# All Options

**Order of precedence: `env var > option > default`**\
**Array environment variables are comma separated: `MY_VAR=thing1,thing2`**

| option                 | required              | env var                 | type                                                                          | description                                                                                                                                                                                                                                                                                                    |
| ---------------------- | --------------------- | ----------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| secret                 | required              | AUTH_SECRET             | string \| string[]                                                            | Provide a secret key or list of keys to use for creating sessions                                                                                                                                                                                                                                              |
| sendCode               | required              |                         | (emailAddress: string, code: string) => Promise\<any>                         | Send a reset code to a specified email address. The user can then copy this code into the reset password form to recover access to their account                                                                                                                                                               |
| store                  | recommended           |                         | [See here](https://github.com/expressjs/session#session-store-implementation) | Provide a store to use for session data. Defaults to an in memory store which is not recommended for a production environment                                                                                                                                                                                  |
| Users                  | recommended           |                         | See above                                                                     | Provide a Users model to store user data. Defaults to an in memory Users model which is not recommended for a production environment                                                                                                                                                                           |
| delay                  | optional              | AUTH_DELAY              | number                                                                        | Milliseconds to delay each request to the auth server.                                                                                                                                                                                                                                                         |
| cors                   | optional              | CORS                    | boolean (string[] if using env var)                                           | Set the cookie properties: `sameSite: true, secure: true`. This will allow cookies to be loaded from any domain and requires the authentication server to be served over https. See the demo folder for an example of serving `https://localhost` using custom certificates. See below for more info           |
| httpOnly               | optional              | HTTP_ONLY               | boolean                                                                       | Whether the cookie willl have the httpOnly flag. True, by default.                                                                                                                                                                                                                                             |
| authBaseUrl            | required for external | AUTH_BASE_URL           | string                                                                        | The external address of your authentication server. This option is required for an identity provider like google or github to redirect users back after signing in but not for local sign in, register or reset password. Can be a full URL or an absolute path, in which case it will use the request origin. |
| google.clientID        | required for google   | GOOGLE_CLIENT_ID        | string                                                                        | Client ID given by [Google.](https://console.developers.google.com/)                                                                                                                                                                                                                                           |
| google.clientSecret    | required for google   | GOOGLE_CLIENT_SECRET    | string                                                                        | Client secret given by [Google.](https://console.developers.google.com/)                                                                                                                                                                                                                                       |
| google.scope           | optional              | GOOGLE_SCOPE            | string[]                                                                      | Scope of information or abilities to be requested by you to the user for their Google account. The 'email' scope is added to all requests in addition to any you specify here                                                                                                                                  |
| github.clientID        | required for github   | GITHUB_CLIENT_ID        | string                                                                        | Client ID given by [Github.](https://github.com/settings/apps)                                                                                                                                                                                                                                                 |
| github.clientSecret    | required for github   | GITHUB_CLIENT_SECRET    | string                                                                        | Client secret given by [Github.](https://github.com/settings/apps)                                                                                                                                                                                                                                             |
| github.scope           | optional              | GITHUB_SCOPE            | string[]                                                                      | Scope of information or abilities to be requested by you to the user for their Github account. The 'user:email' scope is added to all requests in addition to any you specify here                                                                                                                             |
| facebook.clientID      | required for facebook | FACEBOOK_CLIENT_ID      | string                                                                        | Client ID given by [Facebook.](https://github.com/settings/apps)                                                                                                                                                                                                                                               |
| facebook.clientSecret  | required for facebook | FACEBOOK_CLIENT_SECRET  | string                                                                        | Client secret given by [Facebook.](https://developers.facebok.com)                                                                                                                                                                                                                                             |
| facebook.scope         | optional              | FACEBOOK_SCOPE          | string[]                                                                      | Scope of information or abilities to be requested by you to the user for their Facebook account. The 'email' scope is added to all requests in addition to any you specify here                                                                                                                                |
| facebook.profileFields | optional              | FACEBOOK_PROFILE_FIELDS | string[]                                                                      | Profile fields to include when authenticating. The 'email' field is added to all requests in addition to any you specify here                                                                                                                                                                                  |

# Setup cors for cross-domain authentication

If your authentication server is on a different domain than your
client, or you have multiple clients logging in to the same
authentication server, you can use the `cors` middleware and the
`cors` option:

```js
import useAuth, { cors } from '@taylorgrinn/auth-server';


const [authMiddleware, authRouter] = useAuth({
  ...,
  cors: true,
});

app.use(cors('https://taydev.org', 'http://localhost:8081'));
```

The `cors` middleware takes in any number of whitelisted domains and
adds the relevant headers to all requests from each of them.

The `cors` option changes the way cookies are set. Specifically, it
makes them available on any domain and makes them `secure`, only
available when the authentication server is served over https. Check
out the `demo` folder to see an example of serving `https://localhost`
with a self-signed certificate.

If the `CORS` environment variable is set, the cors options will be
set to true and the whitelist will be set to the comma-separated list
value of the `CORS` environment variable:

```sh
# .env file (or however you set env vars)
CORS=https://taydev.org,http://localhost:8081
```

```js
/**
 * You still have to use the cors middleware but the whitelist
 * will be supplied by the environment variable
 */
app.use(cors());
```
