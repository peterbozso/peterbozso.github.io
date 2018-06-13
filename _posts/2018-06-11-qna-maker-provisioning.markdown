---
layout: post
title:  "Provisioning a QnA Maker knowledge base programmatically"
date:   2018-06-11 12:00:00 +0200
categories: qna-maker
---
*This blog post is about the new Global Availability version of the QnA Maker service. You can read the announcement **[at the Bot Framework blog.][qna-ga-announcement]***

During the past week, while working with one of my customers, I faced a challenge that seemed easy at first glance but proved to be a much harder nut to crack when I started to look into the details. I spent the good portion of a day figuring these stuff out, so I hope I can spare that amount of time for you by writing this blog post.

The challenge was the following: I needed to provision a QnA Maker service instance, create a knowledge base on top of it and then publish it, doing all this in an automated way. Sounds easy, right? Actually, it is: we have ARM templates, nice API-s, all those good stuffs. The big challenge was that I had to make sure that the language analyzers of the underlying Azure Search are set correctly to the language of the knowledge base. This is where things got a little bit complicated.

But let's not get ahead of ourselves and start at the beginning!

## The business reason

The QnA Maker is a nice service built with non-technical users in mind. On the QnA Maker portal, these users can go ahead and create their own knowledge bases and train their own services without understanding the details of how the underlying intelligence itself works.

As part of a bigger solution, using other Azure services, that's exactly what we wanted to use the QnA Maker for. But we also wanted to make sure to present the users with a knowledge base seeded with some default content when they first open the portal, thus minimizing the chance of these users accidentally creating a knowledge base with incorrect settings.

### Why all this hassle? What could go wrong?

To see clearly the core of the problem, you first need to read [this page of the docs.][qna-language-support]
These are the key pieces of information from there:
> QnA Maker supports knowledge base content in many languages. However, each QnA Maker service should be reserved for a single language. The first knowledge base created targeting a particular QnA Maker service sets the language of that service.
>
> ...
>
> The language is automatically recognized from the content of the data sources being extracted.
>
> ...
>
> This language cannot be changed once the resource is created.

What this ultimately means is that you either get the language setting right at the very beginning or you are screwed.

## The solution

These are the steps that you need to follow if you want to be 100% sure that language of the underlying service of your knowledge base will be set correctly as part of a fully programmatic deployment.

### 1. Create the QnA Maker service with an ARM template

Azure Resource Manager templates are pretty cool. You can describe the services and the relationships between those services that you want to deploy to Azure in a totally readable (and versionable!) JSON file and with a simple command deploy all of them in one step. You can read more about [Azure Resource Manager and ARM templates here.][arm-overview]

The easiest way to get such an ARM template for a QnA Maker Service is to first deploy one manually to Azure and [export the template][arm-export] from the resource group that encapsulates it.

### 2. Create a knowledge base with dummy data via the API

This was the bit which took most of my time to figure out. As the documentation states (see quote above), when you first create a knowledge base for a QnA Maker service, it will try to figure out the language of the knowledge base and set the analyzer's language in the underlying Azure Search accordingly. What I discovered while playing around with the API is that this language recognition works reliable only for "big enough" data sets of question and answer pairs. It works for a single pair in the rarest of cases, mostly just falling back to English which is obviously not acceptable at all.

So let's say you create a knowledge base with a single QnA pair using [the appropriate API][qna-api-create], with the HTTP request's body looking like this:

{% highlight json %}
{
  "name": "Italian QnA Maker KB",
  "qnaList": [
    {
      "id": 0,
      "answer": "ciao",
      "source": "Editorial",
      "questions": [ "ciao", "buongiorno", "buona sera" ]
    }
  ]
}
{% endhighlight %}

No matter how Italian it looks to you, QnA Maker will happily set the underlying Azure Search's analyzers to English.

The way we can fix this is by feeding QnA Maker a "big enough" set of data initially and then it will pick up it's language correctly. I really don't have a concrete number of question and answer pairs with which it will work correctly. My workaround for this was (somewhat ironically :)) to use the FAQ page of QnA Maker itself, always changing it's language accordingly. So in case we want to create a knowledge base and want to make super sure that the underlying service's language will be set to Italian, we call the API with the following body:

{% highlight json %}
{
  "name": "Italian QnA Maker KB",
  "urls": [ "https://azure.microsoft.com/it-it/services/cognitive-services/qna-maker/faq/" ]
}
{% endhighlight %}

The only thing we change between languages is the culture code in the URL. In the above example it's "it-it". The same part for Hungarian would be "hu-hu" and so on.

### 3. Replace the knowledge base with the correct default data via the API

If we created the knowledge base with the above data, the analyzers will be set correctly, but the first users to start using it will find a lot of question and answer pairs that they have nothing to do with. (Except if they want to create a chatbot for the QnA Maker itself, but that's a very-very rare case among my customer's users.)

To fix this, we need to replace the previously created knowledge base's content (calling [the appropriate REST API][qna-api-replace]) with something more consumable, like this:

{% highlight json %}
{
  "qnaList": [
    {
      "id": 0,
      "answer": "ciao",
      "source": "Editorial",
      "questions": [ "ciao", "buongiorno", "buona sera" ]
    }
  ]
}
{% endhighlight %}

### 4. (Optional) Publish the knowledge base via the API

This step is optional and depends on which state you plan to hand over the freshly created knowledge base to your users. In our case - as I mentioned before - QnA Maker was just part of a bigger solution, so we wanted to make sure that every part of it available before giving the whole thing to the users to play around with.

Luckily, this part is very easy. All you need to do is just to POST [the publish knowledge base endpoint][qna-api-publish].

## Conclusion

To wrap all this in a nice and concise way, you can easily write a small script that deploys the ARM template via the Azure Resource Manager API and then call the above three QnA Maker API-s in order. A great starting point for such a utility script could be the exported ARM template package, since that already contains not just the ARM template itself, but a couple of lines of code in different languages/command line tools (currently Azure CLI, PowerShell, C#, Ruby). Working on top of that it's easy to implement calling the QnA Maker API-s with the correct seed values.

After all is said and done, you can provide your users a QnA Maker environment which you can be sure that is set up correctly to support their choice of language.

[qna-ga-announcement]: https://blog.botframework.com/2018/05/07/announcing-general-availability-of-qnamaker/
[qna-language-support]: https://docs.microsoft.com/en-us/azure/cognitive-services/qnamaker/how-to/language-knowledge-base
[arm-overview]: https://docs.microsoft.com/en-us/azure/azure-resource-manager/
[arm-export]: https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-export-template
[qna-api-create]: https://westus.dev.cognitive.microsoft.com/docs/services/5a93fcf85b4ccd136866eb37/operations/5ac266295b4ccd1554da75ff
[qna-api-replace]: https://westus.dev.cognitive.microsoft.com/docs/services/5a93fcf85b4ccd136866eb37/operations/knowledgebases_publish
[qna-api-publish]: https://westus.dev.cognitive.microsoft.com/docs/services/5a93fcf85b4ccd136866eb37/operations/5ac266295b4ccd1554da75fe