# How to control the utilisation of Watson Assistant 

## Problem
You want to put a Watson Assistant chatbot web widget on a public facing website but: 

* Billing for watson assistant is based on number of different users using it each month. 
* You have no idea how popular it will be. You don't suddenly want a bill for 1,000,000 users.

or 
* You want to sell your service and need to be able to control costs or sell tiered capacity eg £100 for 50 users, £200 for 200 users etc. 

## Solution
Luckily, although there is no built-in way to limit consumption, there is a way to stop billable calls to the Watson Assistant service after a threshold is reached. Watson Assistant allows webhook calls in three ways. 

1. The PRE webhook is called before Watson Assistant processes any input
2. The POST webhook is called after Watson Assistant processes any input and before the result is sent back
3. 

## High Level Steps

The solution is basically to create an API that you call each time a new Watson Assistant session is started. Calls to the API record the session ID in a database and return either sucess or failure based on weather the number of unique session IDs has reached a threshold. Then reset the database or count each month.

For this exercise we'll create the API in NodeRED, secure it with basic authentication and use a Cloudant database to store the session IDs.  

First some notes on Watson Assistant billing/sessions:
Watson Assistant can be called via your own code in a custom app, via various services like Facebook messenger or Slack or via an easy to embed "Web Widget". We will concentrate on the web widget way here and use that in the PoC below.

To maintain state and allow for correct billing, be deafult, the web widget creates a unique session ID for each user and that session ID is stored in a cookie on the browser. In ths way you wont get billed twice if the same user (browser) is used to access your chatbot within a month. That perosn can use the chatbot as much as they want and you will get billed for one user only. But this access method is essentially anoymous and because of this fact there is no way to track the same user accross devices or indeed different browsers on the same device.
If a user is signed in in some way however, the the session ID can be set by you and used accross devices/broswers. In this case you will only be charged for one user no matter what devices or browsers they use. See [here]https://cloud.ibm.com/docs/assistant?topic=assistant-services-information#services-information-user-based-plans for details, and [here]https://www.ibm.com/cloud/watson-assistant/pricing/ for an overview (in the FAQ at the bottom of the page).


1. Create a [Watson Assistant]https://cloud.ibm.com/catalog/services/watson-assistant service in IBM Cloud
2. Create a [Node-RED]https://cloud.ibm.com/developer/appservice/create-app?starterKit=59c9d5bd-4d31-3611-897a-f94eea80dc9f&defaultLanguage=undefined application in IBM Cloud (this wil also create a Cloudant service by default that we reuse to host the database we need)
3. Secure the HTTP endpoints for NodeRED [(instructions)]https://nodered.org/docs/user-guide/runtime/securing-node-red - see HTTP Node Security section
4. Create an API in NodeRED 
5. Create a basic dialogue in Waston Assistant
6. Connect the "Pre" webhook available in Watson Assistant to the API created above
7. Get the dialogue to call the API from the inital "Welcome" node in the dialogue
8. Take action in the dialogue based on the success or failure returned from the API call

