# Building an Express.js API with Zoom Server-to-Server OAuth Authentication

When it comes to building applications on the Zoom platform, Zoom provides a variety of options to best suit your application's needs. 

![Screen Shot 2022-09-20 at 1 20 22 PM](https://user-images.githubusercontent.com/81645097/191358210-fc012523-2bb0-490e-a090-c736b47ee556.png)

If you take a second to glance over the various app types listed above, you'll see that we are deprecating **JWT** apps in favor of the newly added **Server-to-Server OAuth** app type. This deprecation will take place **June, 2023**. If you're interested to read more about why we're doing this, please see our [JWT Deprecation FAQ](https://marketplace.zoom.us/docs/guides/build/jwt-app/jwt-faq/).

To assist Zoom users with this migration, we will create an Express.js API from scratch and integrate it with Zoom via Server-to-Server OAuth. This application will:

-------------------------------------------------------------------------------------------------------
* Spin up an Express.js web server                                                                          
* Generate Zoom Server-to-Server OAuth tokens                                                               
* Store the tokens in-memory (valid 1hr) with Redis                                                         
* Implement a custom middleware to check if the token has expired                                           
  * If expired, automatically generate a new token (no user action required)                                
  * If valid (not expired), pull token from Redis to avoid creating a new token for every request
* Run in a docker container via docker-compose
* Implement the following API's to connect with Zoom's API's

| Users | Method | Endpoint                      | Description          |
|-------|--------|-------------------------------|----------------------|
|       | get    | /api/users                    | list users           |
|       | post   | /api/users/add                | create users         |
|       | get    | /api/users/:userId            | get a user           |
|       | get    | /api/users/:userId/settings   | get user settings    |
|       | patch  | /api/users/:userId/settings   | update user settings |
|       | patch  | /api/users/:userId            | update a user        |
|       | delete | /api/users/:userId            | delete a user        |
|       | get    | /api/users/:userId/meetings   | list meetings        |
|       | get    | /api/users/:userId/webinars   | list webinars        |
|       | get    | /api/users/:userId/recordings | list all recordings  |

| Meetings | Method | Endpoint                                     | Description                       |
|----------|--------|----------------------------------------------|-----------------------------------|
|          | get    | /api/meetings/:meetingId                     | get a meeting                     |
|          | post   | /api/meetings/:userId                        | create a meeting                  |
|          | patch  | /api/meetings/:meetingId                     | update a meeting                  |
|          | delete | /api/meetings/:meetingId                     | delete a meeting                  |
|          | get    | /api/meetings/:meetingId/report/participants | get meeting participation reports |
|          | delete | /api/meetings/:meetingId/recordings          | delete meeting recordings         |

| Webinars | Method | Endpoint                                     | Description                     |
|----------|--------|----------------------------------------------|---------------------------------|
|          | get    | /api/webinars/:webinarId                     | get a webinar                   |
|          | post   | /api/webinars/:userId                        | create a webinar                |
|          | delete | /api/webinars/:webinarId                     | delete a webinar                |
|          | patch  | /api/webinars/:webinarId                     | update a webinar                |
|          | get    | /api/webinars/:webinarId/registrants         | list webinar registrants        |
|          | put    | /api/webinars/:webinarId/registrants/status  | update registrant's status      |
|          | get    | /api/webinars/:webinarId/report/participants | get webinar participant reports |
|          | post   | /api/webinars/:webinarId/registrants         | add a webinar registrant        |

-------------------------------------------------------------------------------------------------------

Let's get started!

## Prerequisites

* Node/npm (https://nodejs.org/en/download/)
* Docker Desktop (https://www.docker.com/products/docker-desktop/)
* Zoom Server-to-Server app credentials (https://marketplace.zoom.us/)
  * **Develop** -> **Build App** -> **Server-to-Server OAuth** create
  * Once created, follow the on-screen directions and add a little bit of information about your app
  * **Scopes**
    * For the purposes of this application, we will add the following scopes:
      * `View and manage all user meetings /meeting:write:admin`
      * `View users information and manage users /user:write:admin`
      * `View and manage all user Webinars /webinar:write:admin`
      * `View and manage all user recordings /recording:write:admin`
      * `View report data /report:read:admin`
    * If you have access to all of these scopes, great! If not, no problem! Modify them in your application accordingly
  * With these scopes selected, navigate back to **App Credentials** and you should see the following pieces of information
    * `Account ID`
    * `Client ID`
    * `Client secret`
  * Keep these handy as we will need them shortly!
  
## Getting Started

Let's create a new directory for our project and begin setting up

```bash
$ mkdir s2s-api
$ cd s2s-api
```

Inside our new project, let's initialize our project with **npm** to create a *package.json* file with all the default options (-y)

```bash
s2s-api$ npm init -y
```

This should create an empty *package.json* file like

```json
{
  "name": "s2s-api",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

Now let's go ahead and populate our *package.json* with the packages we'll need

**Core Dependencies**
```bash
s2s-api$ npm i axios cors dotenv express query-string redis
```

**Dev Dependencies**
```bash
s2s-api$ npm i -D eslint eslint-config-airbnb-base eslint-plugin-import nodemon
```

If any of these packages look foreign to you, please take a minute to look them up individually

**Note**: eslint dependencies are optional, however, I would recommend following some set of linting rules to maintain code structure and readability

Package versions may differ depending on the time of install, but your *package.json* should now look something like

```json
{
  "name": "s2s-api",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "axios": "^0.27.2",
    "cors": "^2.8.5",
    "dotenv": "^16.0.2",
    "express": "^4.18.1",
    "query-string": "^7.1.1",
    "redis": "^4.3.1"
  },
  "devDependencies": {
    "eslint": "^8.23.1",
    "eslint-config-airbnb-base": "^15.0.0",
    "eslint-plugin-import": "^2.26.0",
    "nodemon": "^2.0.20"
  }
}
```

NPM has already defined the entrypoint of our application to *index.js* so let's go ahead and create that file

```bash
s2s-api$ touch index.js
```

Create a *.env* file at the top level of the directory to hold our Zoom credentials

```bash
s2s-api$ touch .env
```

Fill in the following values

```text
ZOOM_ACCOUNT_ID=
ZOOM_CLIENT_ID=
ZOOM_CLIENT_SECRET=
```

**Note**: we never want our credentials to be exposed online, so let's go ahead and add our *.env* to a *.gitignore* file

```bash
s2s-api$ touch .gitignore
```

Inside our *.gitignore*, add the following values to ensure we're not tracking node_modules and user credentials

```text
node_modules
.env
```

**Note**: If you opted out of using a linter, feel free to skip this next step

Create a *.eslintrc.json* at the top level of the directory

```bash
s2s-api$ touch .eslintrc.json
```

Fill in our linting file with the following configurations (or whatever configurations suit you)

```json
{
    "env": {
        "browser": true,
        "commonjs": true,
        "es2021": true
    },
    "extends": "airbnb-base",
    "overrides": [
    ],
    "parserOptions": {
        "ecmaVersion": "latest"
    },
    "rules": {
        "camelcase": "off",
        "no-console": "off"
    }
}
```

Let's take a look at what our project structure should look like at this stage

```
|-- .env
|-- .eslintrc.json
|-- .gitignore
|-- index.js
|-- package.json
```

For the sake of time, let's go ahead and create the remaining directories at once. Feel free to use the command line or editor of your choice to mimic the following structure. Don't worry, we will cover everything shortly.

```
|-- configs/
|--|-- redis.js
|-- constants/
|--|-- index.js
|-- middlewares/
|--|-- tokenCheck.js
|-- routes/
|--|-- api/
|--|--|-- meetings.js
|--|--|-- users.js
|--|--|-- webinars.js
|-- utils/
|--|-- errorHandler.js
|--|-- token.js
|-- .env
|-- .eslintrc.json
|-- .gitignore
|-- index.js
|-- package.json
```

Let's start building our API!

## Implementation

For each file in the project, I will include the **commented code** and **key takeaways** to sum up what each file is doing

#### index.js
```javascript
// index.js

// gives us access to our Zoom credentials via process.env object
require('dotenv').config();

// require imports
const express = require('express');
const cors = require('cors');
const { debug } = require('node:console');

/**
* redis: we will be using redis to store our Zoom Server-to-Server OAuth token in-memory
*
* tokenCheck: middeleware function to be applied to all API routes
*/
const redis = require('./configs/redis');
const { tokenCheck } = require('./middlewares/tokenCheck');

// express app instantiation
const app = express();

// IIFE to connect to redis client
(async () => {
  await redis.connect();
})();

// STDOUT statement on success of redis connection
redis.on('connect', (err) => {
  if (err) {
    console.log('Could not establish connection with redis');
  } else {
    console.log('Connected to redis successfully');
  }
});

// add common express middlewares
app.use([
  cors(),
  express.json(),
  express.urlencoded({ extended: false }),
]);

app.options('*', cors());

/**
 * API Routes w/ tokenCheck middleware
 */
app.use('/api/users', tokenCheck, require('./routes/api/users'));
app.use('/api/meetings', tokenCheck, require('./routes/api/meetings'));
app.use('/api/webinars', tokenCheck, require('./routes/api/webinars'));

// port for our application to run
const PORT = process.env.PORT || 8080;

// run express server
const server = app.listen(PORT, () => console.log(`Listening on port ${[PORT]}!`));

/**
 * graceful shutdown, removes access_token from redis
 */
const cleanup = async () => {
  debug('\nClosing HTTP server');
  await redis.del('access_token');
  server.close(() => {
    debug('\nHTTP server closed');
    redis.quit(() => process.exit());
  });
};

process.on('SIGTERM', cleanup);
process.on('SIGINT', cleanup);
```

### Key takeaways: *index.js*

* Setting up our Express application
* Connecting to redis for token management
* Defining API routes
* Incorporating middlewares
* Graceful shutdown of Express server on exit

***

#### configs/redis.js
```javascript
// configs/redis.js

const { createClient } = require('redis');

// Socket required for node redis <-> docker-compose connection
const Redis = createClient({ socket: { host: 'redis', port: 6379 } });

module.exports = Redis;
```

### Key takeaways: *configs/redis.js*

* Creating a redis client in a way where it can be used in various files throughout the application
* host: redis -> "redis" service name from docker-compose.yml which we will build later
* port: default port used by redis, can be changed of course

***

#### constants/index.js
```javascript
// constants/index.js

const ZOOM_OAUTH_ENDPOINT = 'https://zoom.us/oauth/token';
const ZOOM_API_BASE_URL = 'https://api.zoom.us/v2';

module.exports = {
  ZOOM_OAUTH_ENDPOINT,
  ZOOM_API_BASE_URL,
};
```

### Key takeaways: *constants/index.js*

* Nothing special happening here, just defining some constants so they can be reused throughout the app easily

***

#### utils/errorHandler.js
```javascript
// utils/errorHandler.js

/**
 * @param {*} error object
 * @param {*} res http response
 * @param {*} customMessage error message provided by route
 * @returns error status with message
 */
const errorHandler = (error, res, customMessage = 'Error') => {
  if (!res) return null;
  const { status, data } = error?.response || {};
  return res.status(status ?? 500).json({ message: data?.message || customMessage });
};

module.exports = errorHandler;
```

### Key takeaways: *utils/errorHandler.js*

* Error handler we will use for all of our API routes
* Returns the appropriate error status and error message if available

***

#### utils/token.js
```javascript
// utils/token.js

const axios = require('axios');
const qs = require('query-string');

const { ZOOM_OAUTH_ENDPOINT } = require('../constants');
const redis = require('../configs/redis');

/**
  * Retrieve token from Zoom API
  *
  * @returns {Object} { access_token, expires_in, error }
  */
const getToken = async () => {
  try {
    const { ZOOM_ACCOUNT_ID, ZOOM_CLIENT_ID, ZOOM_CLIENT_SECRET } = process.env;

    const request = await axios.post(
      ZOOM_OAUTH_ENDPOINT,
      qs.stringify({ grant_type: 'account_credentials', account_id: ZOOM_ACCOUNT_ID }),
      {
        headers: {
          Authorization: `Basic ${Buffer.from(`${ZOOM_CLIENT_ID}:${ZOOM_CLIENT_SECRET}`).toString('base64')}`,
        },
      },
    );

    const { access_token, expires_in } = await request.data;

    return { access_token, expires_in, error: null };
  } catch (error) {
    return { access_token: null, expires_in: null, error };
  }
};

/**
  * Set zoom access token with expiration in redis
  *
  * @param {Object} auth_object
  * @param {String} access_token
  * @param {int} expires_in
  */
const setToken = async ({ access_token, expires_in }) => {
  await redis.set('access_token', access_token);
  await redis.expire('access_token', expires_in);
};

module.exports = {
  getToken,
  setToken,
};
```

### Key takeaways *utils/token.js*

* Helper function that retrieves a Zoom Server-to-Server OAuth token
* Helper function that sets token in Redis with an expiration date

***

#### middlewares/tokenCheck.js
```javascript
// middlewares/tokenCheck.js

const redis = require('../configs/redis');
const { getToken, setToken } = require('../utils/token');

/**
  * Middleware that checks if a valid (not expired) token exists in redis
  * If invalid or expired, generate a new token, set in redis, and append to http request
  */
const tokenCheck = async (req, res, next) => {
  const redis_token = await redis.get('access_token');

  let token = redis_token;

  /**
    * Redis returns:
    * -2 if the key does not exist
    * -1 if the key exists but has no associated expire
    */
  if (!redis_token || ['-1', '-2'].includes(redis_token)) {
    const { access_token, expires_in, error } = await getToken();

    if (error) {
      const { response, message } = error;
      return res.status(response?.status || 401).json({ message: `Authentication Unsuccessful: ${message}` });
    }

    setToken({ access_token, expires_in });

    token = access_token;
  }

  // append token to request header -- super important!!
  req.headerConfig = {
    headers: {
      Authorization: `Bearer ${token}`,
    },
  };
  return next();
};

module.exports = {
  tokenCheck,
};
```

### Key takeaways *middlewares/tokenCheck.js*

* Arguably the most important file!
* This middleware function gets applied to all our API routes and checks if a valid token exists in Redis
  * If the token exists, append the token as part of the request header for each route to use
  * If the token doesn't exist or is expired, automatically generate a new token and set it in redis
* The idea here is that Zoom authentication is happening automatically without any user interaction
* In a real world application, you would, of course, want to incorporate user login in front of everything so that only authenticated users of your system get access to these API's.

***
