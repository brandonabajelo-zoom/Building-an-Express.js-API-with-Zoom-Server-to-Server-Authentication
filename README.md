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

Inside the "scripts" object, lets add a script that we'll use later to run the app in development mode

```json
"scripts": {
 "dev": "nodemon index.js"
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
|-- .dockerignore
|-- .env
|-- .eslintrc.json
|-- .gitignore
|-- docker-compose.yml
|-- Dockerfile
|-- index.js
|-- package.json
```

Let's start building our API!

## Implementation

For each file in the project, I will include the **commented code** and a **summary** to explain what each file is doing

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

### Summary: *index.js*

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

### Summary: *configs/redis.js*

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

### Summary: *constants/index.js*

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

### Summary: *utils/errorHandler.js*

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

### Summary: *utils/token.js*

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

### Summary: *middlewares/tokenCheck.js*

* Arguably the most important file!
* This middleware function gets applied to all our API routes and checks if a valid token exists in Redis
  * If the token exists, append the token as part of the request header for each route to use
  * If the token doesn't exist or is expired, automatically generate a new token and set it in redis
* The idea here is that Zoom authentication is happening automatically without any user interaction
* In a real world application, you would, of course, want to incorporate user login in front of everything so that only authenticated users of your system get access to these API's.

***

#### routes/api/users.js
```javascript
// routes/api/users.js

const express = require('express');
const axios = require('axios');
const qs = require('query-string');

const errorHandler = require('../../utils/errorHandler');
const { ZOOM_API_BASE_URL } = require('../../constants');

const router = express.Router();

/**
 * List users
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/users
 */
router.get('/', async (req, res) => {
  const { headerConfig } = req;
  const { status, next_page_token } = req.query;

  try {
    const request = await axios.get(`${ZOOM_API_BASE_URL}/users?${qs.stringify({ status, next_page_token })}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, 'Error fetching users');
  }
});

/**
 * Create users
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/userCreate
 */
router.post('/add', async (req, res) => {
  const { headerConfig, body } = req;

  try {
    const request = await axios.post(`${ZOOM_API_BASE_URL}/users`, body, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, 'Error creating user');
  }
});

/**
 * Get a user
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/user
 */
router.get('/:userId', async (req, res) => {
  const { headerConfig, params, query } = req;
  const { userId } = params;
  const { status } = query;

  try {
    const request = await axios.get(`${ZOOM_API_BASE_URL}/users/${userId}?${qs.stringify({ status })}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error fetching user: ${userId}`);
  }
});

/**
 * Get user settings
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/userSettings
 */
router.get('/:userId/settings', async (req, res) => {
  const { headerConfig, params } = req;
  const { userId } = params;

  try {
    const request = await axios.get(`${ZOOM_API_BASE_URL}/users/${userId}/settings`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error fetching settings for user: ${userId}`);
  }
});

/**
 * Update user settings
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/userSettingsUpdate
 */
router.patch('/:userId/settings', async (req, res) => {
  const { headerConfig, params, body } = req;
  const { userId } = params;

  try {
    const request = await axios.patch(`${ZOOM_API_BASE_URL}/users/${userId}/settings`, body, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error updating settings for user: ${userId}`);
  }
});

/**
 * Update a user
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/userUpdate
 */
router.patch('/:userId', async (req, res) => {
  const { headerConfig, params, body } = req;
  const { userId } = params;

  try {
    const request = await axios.patch(`${ZOOM_API_BASE_URL}/users/${userId}`, body, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error updating user: ${userId}`);
  }
});

/**
 * Delete a user
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/userDelete
 */
router.delete('/:userId', async (req, res) => {
  const { headerConfig, params, query } = req;
  const { userId } = params;
  const { action } = query;

  try {
    const request = await axios.delete(`${ZOOM_API_BASE_URL}/users/${userId}?${qs.stringify({ action })}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error deleting user: ${userId}`);
  }
});

/**
 * List meetings
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/meetings
 */
router.get('/:userId/meetings', async (req, res) => {
  const { headerConfig, params, query } = req;
  const { userId } = params;
  const { next_page_token } = query;

  try {
    const request = await axios.get(`${ZOOM_API_BASE_URL}/users/${userId}/meetings?${qs.stringify({ next_page_token })}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error fetching meetings for user: ${userId}`);
  }
});

/**
 * List webinars
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/webinars
 */
router.get('/:userId/webinars', async (req, res) => {
  const { headerConfig, params, query } = req;
  const { userId } = params;
  const { next_page_token } = query;

  try {
    const request = await axios.get(`${ZOOM_API_BASE_URL}/users/${userId}/webinars?${qs.stringify({ next_page_token })}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error fetching webinars for user: ${userId}`);
  }
});

/**
 * List all recordings
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/recordingsList
 */
router.get('/:userId/recordings', async (req, res) => {
  const { headerConfig, params, query } = req;
  const { userId } = params;
  const { from, to, next_page_token } = query;

  try {
    const request = await axios.get(`${ZOOM_API_BASE_URL}/users/${userId}/recordings?${qs.stringify({
      from,
      to,
      next_page_token,
    })}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error fetching recordings for user: ${userId}`);
  }
});

module.exports = router;
```

#### routes/api/meetings.js
```javascript
// routes/api/meetings.js
const express = require('express');
const axios = require('axios');
const qs = require('query-string');

const errorHandler = require('../../utils/errorHandler');
const { ZOOM_API_BASE_URL } = require('../../constants');

const router = express.Router();

/**
 * Get a meeting
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/meeting
 */
router.get('/:meetingId', async (req, res) => {
  const { headerConfig, params } = req;
  const { meetingId } = params;

  try {
    const request = await axios.get(`${ZOOM_API_BASE_URL}/meetings/${meetingId}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error fetching meeting: ${meetingId}`);
  }
});

/**
 * Create a meeting
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/meetingCreate
 */
router.post('/:userId', async (req, res) => {
  const { headerConfig, params, body } = req;
  const { userId } = params;

  try {
    const request = await axios.post(`${ZOOM_API_BASE_URL}/users/${userId}/meetings`, body, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error creating meeting for user: ${userId}`);
  }
});

/**
 * Update a meeting
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/meetingUpdate
 */
router.patch('/:meetingId', async (req, res) => {
  const { headerConfig, params, body } = req;
  const { meetingId } = params;

  try {
    const request = await axios.patch(`${ZOOM_API_BASE_URL}/meetings/${meetingId}`, body, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error updating meeting: ${meetingId}`);
  }
});

/**
 * Delete a meeting
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/meetingDelete
 */
router.delete('/:meetingId', async (req, res) => {
  const { headerConfig, params } = req;
  const { meetingId } = params;

  try {
    const request = await axios.delete(`${ZOOM_API_BASE_URL}/meetings/${meetingId}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error deleting meeting: ${meetingId}`);
  }
});

/**
 * Get meeting participant reports
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/reportMeetingParticipants
 */
router.get('/:meetingId/report/participants', async (req, res) => {
  const { headerConfig, params, query } = req;
  const { meetingId } = params;
  const { next_page_token } = query;

  try {
    const request = await axios.get(`${ZOOM_API_BASE_URL}/report/meetings/${meetingId}/participants?${qs.stringify({
      next_page_token,
    })}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error fetching participants for meeting: ${meetingId}`);
  }
});

/**
 * Delete meeting recordings
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/recordingDelete
 */
router.delete('/:meetingId/recordings', async (req, res) => {
  const { headerConfig, params, query } = req;
  const { meetingId } = params;
  const { action } = query;

  try {
    const request = await axios.delete(`${ZOOM_API_BASE_URL}/meetings/${meetingId}/recordings?${qs.stringify({ action })}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error deleting recordings for meeting: ${meetingId}`);
  }
});

module.exports = router;
```

#### *routes/api/webinars.js*
```javascript
// routes/api/webinars.js

const express = require('express');
const axios = require('axios');
const qs = require('query-string');

const errorHandler = require('../../utils/errorHandler');
const { ZOOM_API_BASE_URL } = require('../../constants');

const router = express.Router();

/**
 * Get a webinar
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/webinar
 */
router.get('/:webinarId', async (req, res) => {
  const { headerConfig, params } = req;
  const { webinarId } = params;

  try {
    const request = await axios.get(`${ZOOM_API_BASE_URL}/webinars/${webinarId}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error fetching webinar: ${webinarId}`);
  }
});

/**
 * Create a webinar
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/webinarCreate
 */
router.post('/:userId', async (req, res) => {
  const { headerConfig, body, params } = req;
  const { userId } = params;

  try {
    const request = await axios.post(`${ZOOM_API_BASE_URL}/users/${userId}/webinars`, body, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error creating webinar for user: ${userId}`);
  }
});

/**
 * Delete a webinar
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/webinarDelete
 */
router.delete('/:webinarId', async (req, res) => {
  const { headerConfig, params } = req;
  const { webinarId } = params;

  try {
    const request = await axios.delete(`${ZOOM_API_BASE_URL}/webinars/${webinarId}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error deleting webinar: ${webinarId}`);
  }
});

/**
 * Update a webinar
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/webinarUpdate
 */
router.patch('/:webinarId', async (req, res) => {
  const { headerConfig, params, body } = req;
  const { webinarId } = params;

  try {
    const request = await axios.patch(`${ZOOM_API_BASE_URL}/webinars/${webinarId}`, body, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error updating webinar: ${webinarId}`);
  }
});

/**
 * List webinar registrants
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/webinarRegistrants
 */
router.get('/:webinarId/registrants', async (req, res) => {
  const { headerConfig, params, query } = req;
  const { webinarId } = params;
  const { status, next_page_token } = query;

  try {
    const request = await axios.get(`${ZOOM_API_BASE_URL}/webinars/${webinarId}/registrants?${qs.stringify({
      status,
      next_page_token,
    })}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error fetching registrants for webinar: ${webinarId}`);
  }
});

/**
 * Update registrant's status
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/webinarRegistrantStatus
 */
router.put('/:webinarId/registrants/status', async (req, res) => {
  const { headerConfig, params, body } = req;
  const { webinarId } = params;

  try {
    const request = await axios.put(`${ZOOM_API_BASE_URL}/webinars/${webinarId}/registrants/status`, body, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, 'Error updating webinar registrant status');
  }
});

/**
 * Get webinar participant reports
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/reportWebinarParticipants
 */
router.get('/:webinarId/report/participants', async (req, res) => {
  const { headerConfig, params, query } = req;
  const { webinarId } = params;
  const { next_page_token } = query;

  try {
    const request = await axios.get(`${ZOOM_API_BASE_URL}/report/webinars/${webinarId}/participants?${qs.stringify({
      next_page_token,
    })}`, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error fetching webinar participants for webinar: ${webinarId}`);
  }
});

/**
 * Add a webinar registrant
 * https://marketplace.zoom.us/docs/api-reference/zoom-api/methods/#operation/webinarRegistrantCreate
 */
router.post('/:webinarId/registrants', async (req, res) => {
  const { headerConfig, params, body } = req;
  const { webinarId } = params;

  try {
    const request = await axios.post(`${ZOOM_API_BASE_URL}/webinars/${webinarId}/registrants`, body, headerConfig);
    return res.json(request.data);
  } catch (err) {
    return errorHandler(err, res, `Error creating registrant for webinar: ${webinarId}`);
  }
});

module.exports = router;
```

### Summary: *routes/api/users.js*, *routes/api/meetings.js*, *routes/api/webinars.js*
* Create user, webinar, meeting api routes
* Note: the headerConfig coming off our request obj (req) -- our middleware provides this
* Note: for post, put, patch requests, we're just passing the full body payload -- in your application you may choose to validate/parse the body

***

### Docker

At this stage, we've implemented all the business logic of this application! Congrats for making it this far. Lets dockerize the application and run it.

#### *Dockerfile*
```bash
FROM node:16-alpine as base

WORKDIR /app
COPY package*.json /
EXPOSE 8080

FROM base as dev
ENV NODE_ENV=development
RUN npm install -g nodemon && npm install
COPY . /
CMD ["nodemon", "index.js"]
```

***

#### *docker-compose.yml*
```yml
version: '3.8'

services:
  redis:
    image: redis
    ports:
      - '6379:6379'

  dev:
    build:
      context: ./
      target: dev
    ports:
      - '8080:8080'
    volumes:
      - .:/app
      - /app/node_modules
    command: npm run dev
    environment:
      NODE_ENV: development
      DEBUG: nodejs-docker-express:*
    depends_on:
      - redis
    restart: always
```

### Summary: *docker-compose.yml*
* Defined two services for docker to run
* redis: official docker redis image, the name of the service "redis" is also what we passed into our *configs/redis.js* as the **host** value
* dev: development instance of our application with hot reloading

#### *.dockerignore*
```docker
node_modules
npm-debug.log
Dockerfile
docker-compose*
.git
.gitignore
.env
.dockerignore
```

## Run the app

To run the app, ensure Docker Desktop is running first. Then execute the following command to launch our application with hot reloading:

```bash
s2s-api$ docker-compose up dev
```

If everything got set up correctly, our API should now be running in a docker container and available to test using Postman! Head over to postman and send a GET request to `localhost:8080/api/users`. 

## Stop the app

```bash
s2s-api$ docker-compose down
```

## Congratulations

We have just developed a fully functional Express.js API that automatically handles Zoom Server-to-Server Oauth tokens! The full list of API's can be found at the beginning of this tutorial. I hope you found this useful and happy coding!
