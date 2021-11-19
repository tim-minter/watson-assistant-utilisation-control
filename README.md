# How to control the utilisation of Watson Assistant 
AKA "IBM Watson Assistant. Limiting the number of MAUs (monthly active users)"

Note this is a *bit* of a hack.
Note the instructions are not quite complete yet so read through them first to make sure they are of use to you as they stand today. Although we create an API we can call, we don't get to the detail of acting on the database and returning real data yet.

## Problem
You want to put a Watson Assistant chatbot web widget on a public facing website but: 

* Billing for Watson assistant is based on number of different users using it each month. 
* You have no idea how popular it will be. You don't suddenly want a bill for 1,000,000 users.

or 
* You want to sell your service and need to be able to control costs or sell tiered capacity eg £100 for 50 users, £200 for 200 users etc. 

## Solution
There is no built-in way to limit consumption yet, but luckily, there is a way to stop billable calls to the Watson Assistant service after a threshold is reached via one of the webhook options available and the fact that a call to Watson Assistant has to be "meaningful" to be billable. Meaningful is defined as follows:

*"A meaningful interaction happens when an end-user sends a message to your assistant and receives a response. Welcome messages at the beginning of a new conversation are not charged."

Watson Assistant allows webhook calls in three ways:

1. The PRE webhook is called before Watson Assistant processes any input
2. The POST webhook is called after Watson Assistant processes any input and before the result is sent back
3. A webhook that can be called from within any dialogue

Logically we can see here that if we call the right webhook in the right way on the "Welcome" node and either allow or don't allow that communication through, then we can control access (and therefore cost).
How do we stop that communication through though? There are two ways:

1. Hide the user input field. Because this is basically just hiding the input field this still allows a hacker or dedicated user to send messages to Watson Assistant if they REALLY want to.
2. Set a context variable in Watson Assistant if the limit has been reached and test for that variable in the dialogue. This is a really nice solution, and if you are able to add a folder under the Welcome node and move all your other nodes to this folder it's a really simple solution. IF you can't or don't want to move your nodes to a top level folder the you can add the test to every node (example shown below) but that could be some considerable work if you already have many nodes and you'd have to remember to do this to EVERY top/first level node.

![image of additional context variable check](https://github.com/tim-minter/watson-assistant-utilisation-control/blob/main/context-variable-check.png)

This is what it would look like if you use the folder method. All your additional nodes would be placed in the "top level" folder just below the Welcome node. In this way, nothing can be called (not even the "Anything Else" node) if the API call has returned true and the $limitReached context variable has been set to true.

![Image of the dialogue UI when the folder method is used](https://github.com/tim-minter/watson-assistant-utilisation-control/blob/main/dialogue-structure.png)

## High Level Steps

The solution is basically to create an API that you call each time a new Watson Assistant session is started. Calls to the API record the session ID in a database and return either success or failure based on whether the number of unique session IDs has reached a threshold. Then reset the database or count each month.

For this exercise we'll create the API in NodeRED, secure it with basic authentication and use a Cloudant database to store the session IDs.  

First some notes on Watson Assistant billing/sessions:
Watson Assistant can be called via your own code in a custom app, via various services like Facebook messenger or Slack or via an easy to embed "Web Widget". We will concentrate on the web widget way here and use that in the PoC below.

To maintain state and allow for correct billing, be default, the web widget creates a unique session ID for each user and that session ID is stored in a cookie on the browser. In this way you wont get billed twice if the same user (browser) is used to access your chatbot within a month. That person can use the chatbot as much as they want and you will get billed for one user only. But this access method is essentially anonymous and because of this fact there is no way to track the same user across devices or indeed different browsers on the same device.
If a user is signed in in some way however, the the session ID can be set by you and used across devices/browsers. In this case you will only be charged for one user no matter what devices or browsers they use. See [here](https://cloud.ibm.com/docs/assistant?topic=assistant-services-information#services-information-user-based-plans) for details, and [here](https://www.ibm.com/cloud/watson-assistant/pricing/) for an overview (in the FAQ at the bottom of the page).


1. Create a [Watson Assistant](https://cloud.ibm.com/catalog/services/watson-assistant) service in IBM Cloud
2. Create a [Node-RED](https://cloud.ibm.com/developer/appservice/create-app?starterKit=59c9d5bd-4d31-3611-897a-f94eea80dc9f&defaultLanguage=undefined) application in IBM Cloud (this will also create a Cloudant service by default that we reuse to host the database we need)
3. Secure the HTTP endpoints for NodeRED [(instructions)](https://nodered.org/docs/user-guide/runtime/securing-node-red) - see HTTP Node Security section
4. Create an API in NodeRED 
5. Create a basic dialogue in Watson Assistant
6. Connect the "Pre" webhook available in Watson Assistant to the API created above
7. Get the dialogue to call the API from the inital "Welcome" node in the dialogue
8. Take action in the dialogue based on the success or failure returned from the API call

## Steps to create this PoC (Proof of Concept)
1. Sign in to or up for an IBM Cloud account. This PoC can be created using using the free or paid versions of the services available in IBM Cloud. Create a [Watson Assistant](https://cloud.ibm.com/catalog/services/watson-assistant) service in IBM Cloud.
2. Create a [Node-RED](https://cloud.ibm.com/developer/appservice/create-app?starterKit=59c9d5bd-4d31-3611-897a-f94eea80dc9f&defaultLanguage=undefined) application in IBM Cloud. Node-RED uses a Cloudant database to host its data so a free Cloudant service will be created as part of this build. If you already have a free Cloudant instance then you can select that to avoid having to create a new paid service.  
3. For a PoC you may choose not to do this step. Secure the HTTP endpoints for NodeRED [(instructions)](https://nodered.org/docs/user-guide/runtime/securing-node-red) - see HTTP Node Security section. The instructions are minimal and the way you actually have do do this when Node-RED is on the cloud is fairly complex. I will create a separate set of instructions on how to do this soon [link](https://github.com/tim-minter/securing-the-node-red-http-nodes-on-ibm-cloud)
4. It is very easy to create an API in Node-RED with just a few of the built-in Nodes. Go to your Node-RED editor created above and log in. Copy [this flow](https://github.com/tim-minter/watson-assistant-utilisation-control/blob/main/simple-api) to your clipboard then click the burger menu at the top right of the editor and click Import then paste the copied flow into the Import box. Click the red Deploy button at the top right of the page.
5. You now have an API live on the internet at https://[addressofyournoderededitor]/apiv1 (note the "apiv1" part of defined in the first node in the flow)
6. You can test this API by calling it using [this flow](https://github.com/tim-minter/watson-assistant-utilisation-control/blob/main/simple-api-call) (import and deploy in the same way as above). Select the Debug pane from the right hand section of the Node-RED editor then click the tag at the left of the Inject node (labelled Call) to inject data into the http request node (and effectively call the API). Note the debug pane will show two entries one from each Debug node as shown below.

![Image of API and API Calling flow in Node-RED](https://github.com/tim-minter/watson-assistant-utilisation-control/blob/main/simple-flow.png)

7. Now before we build the actual API code we can do the clever stuff with the Welcome node in Watson Assistant. You can skip step 8 to 15 if you want by importing a prepared skill located [here](https://github.com/tim-minter/watson-assistant-utilisation-control/blob/main/skill-UsageLimitEnabledChat.json) into Watson Assistant. Otherwise open your instance of the Watson Assistant service and go to "My first assistant" or create a new assistant. Then click on My first skill (or create a new skill).
8. In the skill, click on Options and then Webooks. In the URL field enter the url to your API created above (https://[addressofyournoderededitor]/apiv1).
9. Click on the Dialog option. You'll notice the two default nodes are shown in the basic dialogue tree. Select the Welcome node and then the "Customise" option and switch on the Callout to webhooks switch (see below)

![Image of call out to webhooks switch location](https://github.com/tim-minter/watson-assistant-utilisation-control/blob/main/callout-to-webhooks.png)

10. Back in the Welcome node set the values you will send with you API call by setting the **key** and **value** pairs as shown in the image below. ```type="checkUsage"```, ```customer_id="customer1"``` and ```metadata="$metadata"```). Click **Add parameter** to add a new key value pair if necessary. This $metadata context variable is created by Watson Assistant and contains the user_id that the assistant has assigned to the user. 
11. Ensure that the **Return variable** field contains ```limit_reached``` (this can contain whatever variable name you want but you must reference the same name in the fields below) and enter ```$limit_reached.result==true``` in the first **If assistant recognises** field and ```Hello. Usage limit has not been reached. How can I help you?``` in the **Respond with** field.
12. Add  ```$limit_reached.result==false``` in the second **If assistant recognises** field (you may need to use the **Add response** link to add a new line) and ```Usage limit has been exceeded, please purchase more capacity.``` in the **Respond with** field.
13. In the third field enter ```anything_else``` and ```Usage limit has been exceeded, please purchase more capacity.``` which catches any other result returned from the API. These lines are shown below.
Note that this ```limit_reached``` variable (like any variable you set in Watson Assistant) remains accessible (by adding the $ symbol in-front of it) to your dialogue for use later.

![Welcome node detail](https://github.com/tim-minter/watson-assistant-utilisation-control/blob/main/welcomenodedetail.png)

14. Close the node and add a folder below the Welcome node. Set the **If assistant recognises** value to ```$limit_reached.result==false``` as shown below.

![Folder setting](https://github.com/tim-minter/watson-assistant-utilisation-control/blob/main/foldersetting.png)

15. We have a dialogue that will call the checkUsage API and enter the folder below that node only if the result is false (and display a welcome message). If the result is true, or anything else, a different message will be displayed and nothing else will happen. 
16. Next we will update the NodeRED flow so that it returns true or false (we can set this manually for now) so we can test this out. Delete your original NodeRED flow and import [this one](https://github.com/tim-minter/watson-assistant-utilisation-control/blob/main/trueFalseFlow.json). It should look like the image below. Remember to **Deploy** the flow in NodeRED (top right red button). When called, this flow will return **false**. If you'd like to return **true** disconnect the link between the **Set to false** node and **API Response** node and add a connection between the **Set to true** and **API Response** node, and remember to **Deploy** the flow in NodeRED (top right red button). 
17. To test this out go back to your Watson Assistant and click the **Try it** button at the top right of the page. This should return "Usage limit has been exceeded, please purchase more capacity." Edit the nodeRED flow as above to return **true** and test again by clicking the **Clear** link at the top off the **Try it out** window.

![Manually set True False Flow ](https://github.com/tim-minter/watson-assistant-utilisation-control/blob/main/trueFalseFlow.png)

18. This is all great and now we need to set up the API so it actually logs the customer_ids and user_ids and returns true or false based on counting these up. 

Next steps TBC
