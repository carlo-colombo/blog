---
title: Serverless Telegram Bot on GC functions
tags: [telegram, gcp, serverless, google cloud functions]
---

I played for some time with the idea of having a telegram bot run _serverless_ in the cloud. Obviously the code run on some server but it is not necessary to care to provision, deploy, starting the application, etc. All you care about is **your code**.

GC Functions can be triggered by **Pub/Sub** events, **buckets** events and **HTTP** invocations. The latter is the one that we are going to provide as webhook to Telegram to be invoked when a message is sent to our bot.

Functions are going to remove some friction from our code, when the request is set with the appropriate `application/json` header the parsed json will be available on the _request_ and when we send back an object is automatically serialized and sent back to the client. 

The example code of the project can be found at https://github.com/carlo-colombo/serverless-telegram-bot-gc-functions

### Prerequisites
* Goole Cloud account and a project. https://cloud.google.com/resource-manager/docs/creating-managing-projects
* Enable Google Cloud Functions and RuntimeConfig API from the API manager.
* Get a telegram bot token, ask it to the [BotFather](https://telegram.me/BotFather).

#### Warning
* Both Google Cloud Functions and RuntimeConfig are both still in beta.
* Even if the [GCP free tier](https://cloud.google.com/free/) is quite extended some costs can be billed.

### The token

```bash
# export for local testing 
export TELEGRAM_TOKEN=133545asdasd

# set the token as GC runtime configuration 
gcloud beta runtime-config configs create prod-config
gcloud beta runtime-config configs variables \
    set telegram/token  "$TELEGRAM_TOKEN" \
    --config-name prod-config
```

### The bot

```js
exports.echoBot = function(req, res){
    const {message:{chat, text}} = req.body
    const echo = `echo: ${text}`

    return getToken()
        .then( token => request.post({
            uri: `https://api.telegram.org/bot${token}/sendMessage`,
            json: true,
            body: {text: echo, chat_id: chat.id}
        }))
        .then(resp => res.send(resp))
        .catch(err => res.status(500).send(err))
}

```
Just an easy bot that echos the received message.

### Retrieving the token

This function return a (promise) of a token either from the runtime config api when run online or from an environment variable when run locally. The value is retrieved using [fredriks/cloud-functions-runtime-config](https://github.com/fredriks/cloud-functions-runtime-config) that wraps the api. `NODE_ENV` is set to production when the function is run online, thus allowing to discriminate in which environment the function run.

```js
function getToken(){
    if (process.env.NODE_ENV == 'production'){
        return require('cloud-functions-runtime-config')
            .getVariable('prod-config', 'telegram/token')
    }
    return Promise.resolve(process.env.TELEGRAM_TOKEN)
}
```

### Local testing

Google provide a local emulator for Functions feature. It allow to local deploy a function to iterate over it without having to deploy to the google server. It reload the code when changed on the file system so it is not necessary to redeploy after the first time.

```
npm -g install @google-cloud/functions-emulator

functions start
functions deploy echoBot --trigger-http

curl -X POST \
  -H "Content-Type: application/json" \
  -d '{
     "message": {
       "chat": {
         "id": 1232456
       },
       "text": "hello world"
     }
   }' \
   http://localhost:8010/litapp-165019/us-central1/echoBot

# To tail logs
watch functions logs read

```

### Deploying

Before deploy the function is required to create a Cloud Storage bucket where the function will be stored

```bash
gsutil mb -c regional -l us-central1 gs://unique-bucket-name

gcloud beta functions deploy function_name \
  --trigger-http \
  --entry-point echoBot \
  --stage-bucket unique-bucket-name
```

### Set up the webhook

Deploying the function with the http trigger will return an url to trigger the function. The url would look like `https://<GCP_REGION>-<PROJECT_ID>.cloudfunctions.net/function_name`. Use this url to set up a web hook for your bot on telegram. You can check more information on webhook on the [Telegram API documentation](https://core.telegram.org/bots/api#setwebhook)

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{
     "url": "https://<GCP_REGION>-<PROJECT_ID>.cloudfunctions.net/function_name"
   }' \
   https://api.telegram.org/bot${TELEGRAM_TOKEN}/setWebhook
```

### Conclusions

Setting up a Telegram bot using Google Cloud Functions is quick and easy, and with the HTTP trigger is possible to seamlessy set a webhook endpoint for a bot without having to care about a server and https certificates (http trigger are https).

One last thing to keep in mind is that Functions are stateless and require to be connected to other services to store data or be for example scheduled. 
