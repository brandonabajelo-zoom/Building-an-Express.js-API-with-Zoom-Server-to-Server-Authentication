# Building an Express.js API with Zoom Server-to-Server OAuth Authentication

When it comes to building applications on the Zoom platform, Zoom provides a variety of options to best suite your application's needs. 

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
* Connect with Zoom's REST API's to retrieve data           
* Run in a docker container via docker-compose                                  
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
s2s-api
|-- .env
|-- .eslintrc.json
|-- .gitignore
|-- index.js
|-- package.json
```

For the sake of time, let's go ahead and create the remaining directories at once. Feel free to use the command line or editor of your choice to mimic the following structure. Don't worry, we will cover everything shortly.

```
s2s-api
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

*index.js*
```javascript
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







    
