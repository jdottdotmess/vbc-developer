---
title: Update Express app for webhook to make API calls to Salesforce
description: In this step you learn how to update the Express application to create Task in Salesforce
---

# Update Express app for webhook to make API calls to Salesforce

In this section, you will update your Express application to create a new Task when your webhook is triggered from a call

To update an ExpressJS application: 

1. In your application, add the [JSForce library](https://jsforce.github.io/) by using `npm install jsforce --save`

2. Create a new file called `.env`. In this file, you will be adding your Salesforce username and password as well as the securtity code, which you should have generated from this [step](_tutorials/en/vonage-integration-platform/log-calls-salesforce/create-webhook-app.md)
The .env will have the following
```javascript
SF_USERNAME=''
SF_PASSWORD=''
SF_TOKEN=''
```
The `SF_USERNAME`  and `SF_PASSWORD` will be the username and password used to login into Salesforce.
The `SF_TOKEN` is the token you should have received via email from the previous step.

2. Create a new Javascript file, called `Salesforce.js` and add the following
```javascript
    var jsforce = require('jsforce');
    var conn = new jsforce.Connection();

    function login() {
        return new Promise((resolve, reject) => {
            conn.login(process.env.SF_USERNAME, process.env.SF_PASSWORD + process.env.SF_TOKEN, function(err, res) {
                if (err) {reject(err); return}
                resolve(res)
            })
        })
    }

    function getContact(phone_number) {
        return new Promise((resolve, reject) => {
            var q = `SELECT Id FROM Contact WHERE Phone='${phone_number}'`
            console.log(q)
            conn.query(q, function(err, res) {
                if (err) {reject(err); console.log(err);}
                resolve(res)
            })
        })
    }

    function createContact(first_name, last_name, phone_number) {
        return new Promise((resolve, reject) => {
            //Create new contact, get the record ID
            console.log(`Create contact ${first_name} ${last_name} ${phone_number}`)
            var data = { FirstName : first_name, LastName : last_name, Phone:phone_number}
            if (first_name != null) {
                data['FirstName'] = first_name
            }
            if (last_name != null) {
                data['LastName'] = last_name
            }
            conn.sobject("Contact").create(data, function(err, ret) {
                if (err || !ret.success) { reject(err);  console.error(err, ret); return }
                console.log("Created new contact id : " + ret.id);
                resolve(ret)
            });
        })
    }

    function addTask(subject, call_dur, recordId) {
        return new Promise((resolve, reject) => {
            conn.sobject("Task").create({ TaskSubtype : 'Call', CallDurationInSeconds : call_dur, Subject:subject, WhoId:recordId}, function(err, ret) {
                if (err || !ret.success) { reject(err);  console.error(err, ret); return }
                console.log("Created new task id : " + ret.id);
                resolve(ret)
            });
        })
    }

    module.exports.login = login
    module.exports.getContact = getContact
    module.exports.createContact = createContact
    module.exports.addTask = addTask
```

4. Update the code in `app.js` to import this new file

```javascript
var salesforce = require('./Salesforce.js')
```
When the app loads, write the code to login using your Salesforce credentials
```javascript
salesforce.login()
.then(function(res) {
  console.log(res)
}).catch((function(err) {
  console.log(err)
}))
```
This code will login to Salesforce using your credentails.


3. Update the code in the `app.post('/webhook`) section of your application.
    
    ```javascript
     const express = require('express')
     const app = express()
     const port = 3000
    app.post('/webhook', (req, res) => {
        var event = req.body.event
        var callerId = event.callerId;
        var first_name = null
        var last_name = null
        var phone_number = event.phoneNumber.replace(/\D/g,'')
        if (typeof(callerId) != "undefined") {
            [last_name, first_name] = callerId.split(" ")
        } else {
            last_name = phone_number
        }
        var direction = event.direction
        var duration = event.duration
        var state = event.state
        if (state == "ANSWERED")  {
            var name = `${first_name} ${last_name}`
            if (typeof(callerId) == "undefined") {
                name = phone_number
            }
            var subject = `${direction.toLowerCase()} call with ${name}`
            salesforce.getContact(phone_number)
            .then(function(contact) {
            console.log(contact)
            if (contact["totalSize"] == 0) {
                //Create new contact
                return salesforce.createContact(first_name, last_name, phone_number)
                .then(function(contact) {
                var contactId = contact["Id"]
                return salesforce.addTask(subject, duration, contactId)
                })
            } else {
                //Grab the first contact
                var contactId = contact["records"][0]["Id"]
                return salesforce.addTask(subject, duration, contactId)
            }
            }).catch(function(err) {
            console.log(err)
            })
        }
        res.sendStatus(200);
        });
     app.listen(port, () => console.log(`Example app listening at http://localhost:${port}`))
    ```

This code will be tiggered when a call is made or received from your VBC number, When the call is completed(`if (state == "ANSWERED")`), the application will first look for a Contact with the given phone number(`event.phoneNumber`).
This will call the `salesforce.getContact()` to search for the Contact. If the Contact exists, we create a new Task using the function `salesforce.addTask()`. This will create a new Task in Salesforce that includes the title, the assoicated Contact(using the `contactId`) and the duration of the call. 
If there are no Contacts that match the given phone number, using the `contact["totalSize"] == 0` check, the application will then create a new Contact using the `event.callerId` property from the webhook, and split the string into a first name and last name. Note, outgoing calls MAY not have this property. In this case, we will just use the phone number as the Contact's last name.

4. To start your application, run the following command:

    `node app.js`

Your application will now create a new Task in Salesforce when a completed call is made or received. 

> Note: Make sure the port you have specified (`300`) is the same port you use when creating your ngrok URL.