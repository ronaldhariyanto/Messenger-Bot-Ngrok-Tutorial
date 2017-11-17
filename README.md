# Facebook Messenger Bot with Ngrok

This repo is forked from [fb-masterclass-jakarta/step-create-bot-messenger](https://github.com/fb-masterclass-jakarta/step-create-bot-messenger) and [jw84/messenger-bot-tutorial](https://github.com/jw84/messenger-bot-tutorial)

---

### WHAT IS NEEDED?

---

- NodeJS
- Facebook Page
- Facebook App
- Webhook Service
- Ngrok

---

### STEP I: BUILD THE SERVER

#### 1. Install NodeJS, NPM and Express

  a. Download and install **NodeJS** from official website: https://nodejs.org/.

  b. Check version.
  ```sh
  node -v
  ```

  c. Install latest **npm**
  ```sh
  npm install npm -g
  ```

  d. Create directory anywhere
  ```sh
  mkdir myBot
  cd myBot
  npm init
  ```

  e. Install Express is for the server, request is for sending out messages and body-parser is to process messages
  ```sh
  npm install express request body-parser --save
  ```


#### 2. Install and Run Ngrok

  a. Download and install **Ngrok** from official website: https://ngrok.com/

  b. Unzip from a terminal to any directory. On Windows, just double click ngrok.zip.
  ```sh
  unzip /path/to/ngrok.zip
  ```

  c. Start the **ngrok**
  ```sh
  ngrok http 5000
  ```


#### 3. Create file index.js

  ```javascript
  'use strict'

  const express = require('express')
  const bodyParser = require('body-parser')
  const request = require('request')
  const app = express()

  app.set('port', (process.env.PORT || 5000))

  // Process application/x-www-form-urlencoded
  app.use(bodyParser.urlencoded({extended: false}))

  // Process application/json
  app.use(bodyParser.json())

  // Index route
  app.get('/', function (req, res) {
    res.send('Hello world, I am a chat bot')
  })

  // for Facebook verification
  app.get('/webhook/', function (req, res) {
    if (req.query['hub.verify_token'] === 'make_indonesian_great_again') {
      res.send(req.query['hub.challenge'])
    }
    res.send('Error, wrong token')
  })

  // Spin up the server
  app.listen(app.get('port'), function() {
    console.log('running on port', app.get('port'))
  })
  ```

#### 4. Run server on index.js

```javascript
node index.js
```

---

### STEP II: SETUP FACEBOOK PAGE AND APP

#### 1. Create Facebook Page

  a. Click https://www.facebook.com/pages/create

#### 2. Create Facebook App

  a. Click https://developers.facebook.com/apps/

  b. Add Product **Facebook Messenger**

  c. Test connection
  ```sh
  curl -X POST "https://graph.facebook.com/v2.6/me/subscribed_apps?access_token=PAGE_ACCESS_TOKEN"
  ```

#### 3. Set up Webhook

  a. Click **Messenger - Settings**

  b. Click **Setup Webhooks**

  c. Add URL from **ngrok**
  ```sh
  https://blabla.ngrok.io
  ```

---

### STEP III: SETUP THE BOT

#### 1. Add AccessToken
```javascript
const token = "<PAGE_ACCESS_TOKEN>"
```

#### 1. Add an API endpoint to index.js to process messages
```javascript
app.post('/webhook/', function (req, res) {
    let messaging_events = req.body.entry[0].messaging
    for (let i = 0; i < messaging_events.length; i++) {
	    let event = req.body.entry[0].messaging[i]
	    let sender = event.sender.id
	    if (event.message && event.message.text) {
		    let text = event.message.text
		    sendTextMessage(sender, "Text received, echo: " + text.substring(0, 200))
	    }
    }
    res.sendStatus(200)
})
```

#### 2. Create function to send text messages
```javascript
function sendTextMessage(sender, text) {
    let messageData = { text:text }
    request({
	    url: 'https://graph.facebook.com/v2.6/me/messages',
	    qs: {access_token:token},
	    method: 'POST',
		json: {
		    recipient: {id:sender},
			message: messageData,
		}
	}, function(error, response, body) {
		if (error) {
		    console.log('Error sending messages: ', error)
		} else if (response.body.error) {
		    console.log('Error: ', response.body.error)
	    }
    })
}
```

---

### STEP IV: AUTHENTICATION

#### 1. Improve Authentication

  a. Add crypto library
  ```javascript
  const crypto = require('crypto')
  ```

  b. Add APP Secret
  ```javascript
  const AppSecret = 'APP_YOUR_SECRET';
  ```

  c. Check first
  ```javascript
  app.use(bodyParser.json({verify: verifyRequestSignature}))
  ```

  d. Add function to verify
  ```javascript
  function verifyRequestSignature(req, res, buf){
    let signature = req.headers["x-hub-signature"];
    
    if(!signature){
      console.error('You dont have signature')
    } else {
      let element = signature.split('=')
      let method = element[0]
      let signatureHash = element[1]
      let expectedHash = crypto.createHmac('sha1', AppSecret).update(buf).digest('hex')

      console.log('signatureHash = ', signatureHash)
      console.log('expectedHash = ', expectedHash)
      if(signatureHash != expectedHash){
        console.error('signature invalid, send message to email or save as log')
      }
    }
  }
  ```

---

### STEP V: IMPROVE

#### 1. Improve code above!

```javascript
app.post('/webhook/', function (req, res) {
  let data = req.body
  if(data.object == 'page'){
    data.entry.forEach(function(pageEntry) {
      pageEntry.messaging.forEach(function(messagingEvent) {
        if(messagingEvent.message.text){
          sendTextMessage(messagingEvent.sender.id,messagingEvent.message.text);
        } else {
          sendTextMessage(messagingEvent.sender.id,'Service Belum Support Untuk Mendeteksi Hal ini');
        }
      }); 
    });
    res.sendStatus(200)
  }
})
```

#### 2. Improve Again!

```javascript
function sendTextMessage(sender, text) {
  let url = `https://graph.facebook.com/v2.6/${sender}?fields=first_name,last_name,profile_pic&access_token=${token}`;
  
  request(url, function (error, response, body) {
    if (!error && response.statusCode == 200) {
      let parseData = JSON.parse(body);
      let messageData = {
        text: `Hi ${parseData.first_name} ${parseData.last_name}, you send message : ${text}`
      }
      request({
        url: 'https://graph.facebook.com/v2.10/me/messages',
        qs: {
          access_token: token
        },
        method: 'POST',
        json: {
          recipient: {
            id: sender
          },
          message: messageData,
        }
      }, function (error, response, body) {
        if (error) {
          console.log('Error sending messages: ', error)
        } else if (response.body.error) {
          console.log('Error: ', response.body.error)
        }
      })
    }
  })
}
```
