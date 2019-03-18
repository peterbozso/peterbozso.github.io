---
layout: post
title:  "Debugging a Microsoft Bot Framework chatbot through the Direct Line API 3.0"
date:   2018-11-02 08:00:00 +0200
---
To put this post in context, let's assume for a minute that you are developing a chatbot with the Microsoft Bot Framework which integrates with some unsupported channels (for example Viber or your very own native mobile application) using the Direct Line API. In this case, there's a very high chance that you'd like to test and debug the bot end-to-end in a test environment (be it on Azure or on your local computer - more about the latter [here][bot-ngrok]) through the Direct Line before deploying it into production. Postman is just the perfect tool for such a task!

Since you are reading this article, I assume that you are more or less familiar with the Direct Line API 3.0 and Postman. If not, you can find the full documentation of the API [here][direct-line-api] and of Postman [here][postman].

## Postman package

To get quickly up to speed, you can find a Postman collection and a default environment I use for my own testings packed together in [this repository][postman-package].

After downloading the package, you can [import the whole thing in one step][postman-import] into Postman by using the Import Folder functionality and choosing the **directline-postman** folder in your freshly cloned local repository.

This neat little package contains four requests that I use frequently for testing the chatbots I develop:

* Start conversation
* Send event
* Send message
* Receive activities

The names of the requests are pretty self-explanatory, especially if you are familiar with the Direct Line API 3.0.

Using those four request types, my usual testing workflow looks like this:

1. Create a new conversation by issuing a **Start conversation** request.
2. Using **Send event** and **Send message** to trigger the dialog/chain of dialogs I'd like to test.
3. Check the conversation history with **Receive activities** to see if everything is okay.

In the above list, sending an event and sending a message are pretty similar, since they are POST-ing the very same endpoint, but with a different body. The reason I included both is because these are the two Activity types I use the most during my testings. Of course, based on these and the [schema's documentation][direct-line-activity], sending your very own activity object is easy-peasy.

This is all pretty basic Postman usage so far. But to get the authentication working and to make the developer experience even more streamlined and less error prone, I used two more advanced Postman features in this package: environments and tests.

## Postman environments

In Postman there's quite a few options to utilize [environments][postman-environment] and connected to that, [variables][postman-variable] in your requests.

In my package I am using environments and environment variables for two things: I store the [Direct Line secret for authentication][direct-line-secret] and the current [conversation's ID][direct-line-conversation-id] as variables to send requests to the bot conveniently after each other.

### Authentication

You can do authentication against the Direct Line in two ways: using the Direct Line secret all the time or generating a new token for each conversation using the Direct Line secret. (See [this link][direct-line-token] for details.)

I choose to go with using the Direct Line secret, since generating a token for every conversation is useful only when you have a client facing application (a website or a mobile application) that needs to talk to your bot through the Direct Line. In that case, you'd generate a token on your back-end using the secret, which never expires, and send the token to the frontend. The front-end will be using only the tokens that have an expiration time, thus protecting your system and your users. For example, the open source [WebChat control][webchat] (which is a reference implementation) supports and highly recommends this authentication method (the token one) as well. More about this [here][webchat-auth].

To use the same Direct Line secret in all the requests, I am using an environment variable which (after importing the folder) you can find in the **Direct Line 3.0** environment. You need to set the value of the **directLineSecret** variable to your own secret. The imported collection (which is named **Direct Line 3.0** as well) then uses this variable to set the **Authorization** header of all the requests contained in it. For more information see the **Inherit auth from parent** section [here][postman-auth].

Of course, I could have simply put the Direct Line secret directly into the collection's auth settings. But it's rare for me to work on only one chatbot project (or only one instance of the same chatbot) at the same time. Doing it this way, through an environment variable, you can easily switch between chatbots by adding a new environment containing a variable named **directLineSecret** with it's value set to the other chatbot's secret. Then you just select the correct environment from the dropdown on the upper right corner of the main Postman window instead of doing the tedious ritual of right click on collection name -> Edit -> Authorization -> replace the Auth token every time you need to switch between bots.

### Conversation ID

If you look at the requests in the collection, you'll see that all of them (except **Start conversation**) have this URL:

>https://directline.botframework.com/v3/directline/conversations/{% raw  %}{{conversationId}}{% endraw  %}/activities

To make the debugging as easy as possible, I choose to store the conversation ID which is returned by the **Start conversation** request in the **conversationId** variable. Then I re-use it in consequent requests, so they will all work in the context of the same conversation. I do this until the next **Start conversation** request, which of course changes the value of the variable.

This brings us to the question of how the **Start conversation** request's response can set the **conversationID** variable. This is when Postman tests come into play.

## Postman tests

From the [Postman documentation about tests][postman-test]:

> A Postman test is essentially JavaScript code executed after the request is sent, allowing access to the **pm.response** object.

In our case, the **pm.response** object is exactly what we need! This will contain the response to the request that is being tested. All we need to do is to mine out the value of the conversation ID from it and store it in the **conversationId** environment variable.

*Of course, the name "test" is a little bit confusing here, because we are not testing anything, we just simply save that piece of information in a variable. But this is the only place in Postman we can do so, there's no dedicated "Post-request Script" category.*

To set the variable, we will use the below JavaScript code as the test of the **Start conversation** request:
```javascript
pm.test("Save conversation ID", () => {
    var jsonData = pm.response.json();
    pm.environment.set("conversationId", jsonData.conversationId);
});
```
This is a pretty trivial piece of code: first we deserialize the content of the response of our request as a JSON object, and then we save it's **conversationId** property as the value of our environment variable.

After this, if you issue any of the other three requests that have the **conversationId** variable in the URL, they will interact with the correct conversation. Also, since we are saving the conversation ID in an environment variable, you can edit its value anytime on the UI of Postman as well. This could come in handy when one of your co-workers or customers sends you a conversation ID and ask you to check the history or do some debugging on that specific conversation.

## Summary

I am using this small Postman package almost daily in my job and it proved to be a great productivity boost to me during the past half year. I hope you'll find it equally useful!

[bot-ngrok]: https://blogs.msdn.microsoft.com/jamiedalton/2016/07/29/ms-bot-framework-ngrok/
[direct-line-api]: https://docs.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-direct-line-3-0-concepts?view=azure-bot-service-4.0
[postman]: https://www.getpostman.com/docs/v6/
[postman-package]: https://github.com/BotBuilderCommunity/botbuilder-community-tools/tree/master/directline-postman
[postman-import]: https://www.getpostman.com/docs/v6/postman/collections/data_formats
[direct-line-activity]: https://docs.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-connector-api-reference?view=azure-bot-service-4.0#activity-object
[postman-environment]: https://www.getpostman.com/docs/v6/postman/environments_and_globals/intro_to_environments_and_globals
[postman-variable]: https://www.getpostman.com/docs/v6/postman/environments_and_globals/variables
[direct-line-secret]: https://docs.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-direct-line-3-0-authentication?view=azure-bot-service-4.0
[direct-line-conversation-id]: https://docs.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-direct-line-3-0-start-conversation?view=azure-bot-service-4.0
[direct-line-token]: https://docs.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-direct-line-3-0-authentication?view=azure-bot-service-4.0
[webchat]: https://github.com/Microsoft/BotFramework-WebChat
[webchat-auth]: https://docs.microsoft.com/en-us/azure/bot-service/bot-service-channel-connect-webchat?view=azure-bot-service-4.0
[postman-auth]: https://www.getpostman.com/docs/v6/postman/sending_api_requests/authorization
[postman-test]: https://www.getpostman.com/docs/v6/postman/scripts/test_scripts