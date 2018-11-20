# Glip Ping Chatbot

This chatbot is super simple, it replies "pong" whenever it receives "ping".

This chatbot is mainly served as a hello-world style demo bot for developers.

This chatbot is powered by the [ringcentral-chatbot framework for JavaScript](https://github.com/tylerlong/ringcentral-chatbot-js).


## Quick start with AWS Lambda

Please note that, even you are planning to host your bot on AWS Lambda, it is still useful to run an Express.js server during development. So you might be interested in [Quick start with Express.js](https://github.com/tylerlong/glip-ping-chatbot/tree/express).


### Install dependencies

```
yarn add ringcentral-chatbot pg serverless-http
yarn add --dev serverless
```

We installed `pg` because we would like to use PostgreSQL as database. In order to deploy to AWS Lambda, you can choose either PostgreSQL or MySQL as database. Install `mysql` of you use MySQL as database.
For local development, you can choose SQLite(refer to [Quick start with Express.js](https://github.com/tylerlong/glip-ping-chatbot/tree/express#quick-start-with-expressjs)).

We use [serverless-http](https://github.com/dougmoscrop/serverless-http) to use Express.js code in AWS Lambda. So that we can reuse code between Express.js and AWS Lambda.

We use [serverless](https://github.com/serverless/serverless) framework to help us deploy the project to AWS  with ease.


### Coding

Create `lambda.js` file with the following content:

```js
const createApp = require('ringcentral-chatbot/dist/apps').default
const { createAsyncProxy } = require('ringcentral-chatbot/dist/lambda')
const serverlessHTTP = require('serverless-http')

const handle = async event => {
  const { type, text, group, bot } = event
  if (type === 'Message4Bot' && text === 'ping') {
    await bot.sendMessage(group.id, { text: 'pong' })
  }
}
const app = createApp(handle)
module.exports.app = serverlessHTTP(app)
module.exports.proxy = createAsyncProxy('app')
```

You may notice that the code is similar to [its Express.js version](https://github.com/tylerlong/glip-ping-chatbot/tree/express#coding), Especially the `handle` method is identical. This is because we would like to allow code reusing, no matter you deploy your chatbot as Express.js server or to AWS Lambda.


### Create a RingCentral app

Please read the instructions for [how to create a RingCentral chatbot app](https://github.com/tylerlong/ringcentral-chatbot-js#create-a-ringcentral-app).

Leave "OAuth Redirect URI" empty for now because we don't know it until we deploy the project to AWS.


### Create & configure database on AWS RDS

Login [AWS Console](https://console.aws.amazon.com) and navigate to [RDS](https://console.aws.amazon.com/rds/home?region=us-east-1)

Create a PostgreSQL database, configure the security group so that it is publicly accessibly with username and password.

Take a note of the database uri, database name, username and password, we will use them soon.


### Specify environment variables:

Create `.env.yml` file using [.lambda.env.yml](https://github.com/tylerlong/ringcentral-chatbot-js/blob/master/.lambda.env.yml) as template.

- RINGCENTRAL_SERVER, use https://platform.dev.ringcentral.com for sandbox and https://platform.ringcentral.com for production
- RINGCENTRAL_CHATBOT_DATABASE_CONNECTION_URI, please sepcify connection URI to a relational database.
    - For
- RINGCENTRAL_CHATBOT_CLIENT_ID & RINGCENTRAL_CHATBOT_CLIENT_SECRET could be found in the newly created RingCentral app.
- RINGCENTRAL_CHATBOT_SERVER We don't know until we deploy the project to AWS, let's specify a dummy one for now: `https://xxxxxx.execute-api.us-east-1.amazonaws.com/prod`


### Create serverless.yml

Create `serverless.yml` with following content:

```yml
service:
  name: demo-ping-chatbot
provider:
  stage: ${opt:stage, 'prod'}
  name: aws
  runtime: nodejs8.10
  region: us-east-1
  memorySize: 256
  environment: ${file(./.env.yml)}
  iamRoleStatements:
    - Effect: Allow
      Action:
        - lambda:InvokeFunction
      Resource: "*"
package:
  exclude:
    - ./**
    - '!dist/**'
    - '!node_modules/**'
  excludeDevDependencies: true
functions:
  app:
    handler: lambda.app
    timeout: 300
  proxy:
    handler: lambda.proxy
    events:
      - http: 'ANY {proxy+}'
```

### Deploy to AWS

```
npx sls deploy
```


### Update .env.yml

Since we just deployed the project to AWS, now we know the RINGCENTRAL_CHATBOT_SERVER. It should be in the form of "https://xxxxxx.execute-api.us-east-1.amazonaws.com/prod"

Update `RINGCENTRAL_CHATBOT_SERVER` in `.env.yml`.


### Deploy to AWS again

Just to apply the change to `.env.yml`:

```
npx sls deploy
```


### Update "OAuth Redirect URI" for RingCentral app

Login https://developer.ringcentral.com, navigate to your bot app, open "Settings" tab, set "OAuth Redirect URI" to "https://xxxxxx.execute-api.us-east-1.amazonaws.com/prod/bot/oauth"



### Create database tables

```
curl -X PUT https://<chatbot-server>/prod/admin/setup-database
```

For more information, please read [setup database](https://github.com/tylerlong/ringcentral-chatbot-js#setup-database).


### [Add the bot to Glip](https://github.com/tylerlong/glip-ping-chatbot/tree/master#add-the-bot-to-glip)


### [Test the bot](https://github.com/tylerlong/glip-ping-chatbot/tree/master#test-the-bot)
