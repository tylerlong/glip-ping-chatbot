# Glip Ping Chatbot

This chatbot is super simple, it replies "pong" whenever it receives "ping".

This chatbot is mainly served as a hello-world style demo bot for developers.

This chatbot is powered by the [ringcentral-chatbot framework for JavaScript](https://github.com/tylerlong/ringcentral-chatbot-js).


### Add the bot to Glip

Login https://developer.ringcentral.com, navigate to your app, go to "Bot" tab, click "Add to Glip" button.

Give the bot a name. (during development, I normally name it the current timestamp, such as "2018-11-20-12-13")

Click "Authorize" button to continue.


### Test the bot

Login https://glip-app.devtest.ringcentral.com/

Find the bot we just added by name. You might need to wait for several minutes. And please logout and re-login if you cannot find the bot user.

Send "ping" to be bot. The bot should reply you with "pong".

If you add the bot to a multiple user chat group, you need to mention the bot when sending it something, otherwise it won't respond.
