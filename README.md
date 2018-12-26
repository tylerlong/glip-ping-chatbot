# Glip Ping Chatbot

This chatbot is super simple, it replies "pong" whenever it receives "ping".

This chatbot is mainly served as a hello-world style demo bot for developers.

This chatbot is powered by the [ringcentral-chatbot framework for JavaScript](https://github.com/tylerlong/ringcentral-chatbot-js).


## Quick start with Express.js

[Watch this video](https://youtu.be/CR66cwHvsOI) and follow the instructions below:


### Create an empty project

```
mkdir <bot-project-name>
cd <bot-project-name>
```


### Install dependencies

```
yarn add ringcentral-chatbot sqlite3
yarn add --dev dotenv ngrok
```

[ringcentral-chatbot](https://github.com/tylerlong/ringcentral-chatbot-js) is the RingCentral chatbot framework.

We use SQLite as our database here, so we installed `sqlite3`. It is a good idea to use SQLite during development phase.
Currently PostgreSQL, MySQL & SQLite are supported. Please install `pg` if you use PostgreSql or `mysql` if you use MySQL.

`dotenv` allows us to specify environment variables in a `.env` file. `ngrok` allows us to generate a public address.


### Start ngrok to get a public address

This step is optional if you have other ways to get your bot a public address. For example, you might have an VPS with public IP address. But ngrok is pretty handy during development phase:

```
./node_modules/.bin/ngrok http -region eu 3000
```

Please take a note of the public address with scheme "https", it should be in the form of `https://xxxxx.eu.ngrok.io`. We will need it soon.

Port 3000 is where we run our bot's Express.js process. I add `-region eu` because it seems to be faster, it is optional.

In content below, we call the public address `https://<chatbot-server>`.


### Create a RingCentral app

Please read the instructions for [how to create a RingCentral chatbot app](https://github.com/tylerlong/ringcentral-chatbot-js#create-a-ringcentral-app).

Redirect URI should be set to `https://<chatbot-server>/bot/oauth`.


### Specify environment variables:

Create `.env` file using [.express.env](https://github.com/tylerlong/ringcentral-chatbot-js/blob/master/.express.env) as template.

- RINGCENTRAL_SERVER, use https://platform.dev.ringcentral.com for sandbox and https://platform.ringcentral.com for production
- RINGCENTRAL_CHATBOT_DATABASE_CONNECTION_URI, please sepcify connection URI to a relational database.
    - It is recommended to use SQLite for dev: sqlite:////absolute-path-to-this-project/db.sqlite
- RINGCENTRAL_CHATBOT_CLIENT_ID & RINGCENTRAL_CHATBOT_CLIENT_SECRET could be found in the newly created RingCentral app.
- RINGCENTRAL_CHATBOT_SERVER is the public address generated by ngrok
    - For produciton, you might put your app behind nginx/apache and use the public address of those HTTP servers.
- RINGCENTRAL_CHATBOT_EXPRESS_PORT is the port we used for Express.js. It should match the ngrok command above.


### coding

Create `express.js` file with following content:

```js
const createApp = require('ringcentral-chatbot/dist/apps').default

const handle = async event => {
  const { type, text, group, bot } = event
  if (type === 'Message4Bot' && text === 'ping') {
    await bot.sendMessage(group.id, { text: 'pong' })
  }
}
const app = createApp(handle)
app.listen(process.env.RINGCENTRAL_CHATBOT_EXPRESS_PORT)
```

For latest code, please check [express.js of this repository](./express.js).


### Start the bot

```
node -r dotenv/config express.js
```


### Create database tables

```
curl -X PUT -u admin:user https://<chatbot-server>/admin/setup-database
```

For more information, please read [setup database](https://github.com/tylerlong/ringcentral-chatbot-js#setup-database).


### [Add the bot to Glip](https://github.com/tylerlong/glip-ping-chatbot/tree/master#add-the-bot-to-glip)


### [Test the bot](https://github.com/tylerlong/glip-ping-chatbot/tree/master#test-the-bot)
