---
title: How an SDK Can Add Value to REST APIs
description: This article provides an inside look into the design of the Java
  SDK's implementation of the Vonage Messages v1 API.
thumbnail: /content/blog/how-an-sdk-can-add-value-to-rest-apis/sdk-rest-apis.png
author: sina-madani
published: true
published_at: 2022-08-04T11:48:15.975Z
updated_at: 2022-08-04T11:48:16.003Z
category: release
tags:
  - java
  - messages-v1
  - messages-api
comments: true
spotlight: false
redirect: ""
canonical: ""
outdated: false
replacement_url: ""
---
The [Vonage Messages v1 API](https://developer.vonage.com/messages/overview) was recently promoted to General Availability status. Following this announcement, the Developer Relations Tooling Team have been hard at work on implementing support for it in our server SDKs. 

Collectively, we decided that each SDK should implement the API in whatever way makes the most sense, given the language, community and general approach used to support other APIs in the SDK, for consistency. Our goal is to ensure that each SDK feels idiomatic, rather than a one-size-fits-all approach where the implementation feels auto-generated. The Messages v1 API is a good way to illustrate this, hence this article. 

Naturally, as the Java Developer Advocate on the team, I'm going to focus on the Java SDK's implementation and describe the rationale behind its design. But first, let's examine the [Messages API specification](https://developer.vonage.com/api/messages-olympus), which is what we need to implement.

The API has a single endpoint, which is to send a message via a POST request. The complexity arises from the schema of messages. There are two main concepts here: *channels* and *message types*. The former refers to the service that is used to send the message (SMS, MMS, WhatsApp, Viber, Facebook Messenger). The latter describes the medium of the message (text, image, audio, video, file etc.). 

Each channel supports a subset of message types. For example, you can't send a video via SMS. Some channels, like WhatsApp, support templates and custom message types, which allow you to send more complex messages or [share your location](https://github.com/Vonage/vonage-java-code-snippets/blob/master/src/main/java/com/vonage/quickstart/messages/whatsapp/SendWhatsappLocation.java).

Furthermore, each channel places unique restrictions on message contents. For example, the length limit on texts varies by channel, as do supported file types for media messages - for instance, WhatsApp supports a wide range of audio formats, whilst Messenger only supports MP3. For media messages, some channels allow an optional text description (referred to as a *caption* in the API specification) to accompany the file, whilst others do not. 

This may even vary within channels - for example, one can include a caption with an MMS, except if it's a vCard (which, by the way, is unique to the MMS channel). From an implementor's perspective, the challenge then is to find a suitably structured and efficient way to capture these requirements whilst minimising confusion to consumers of the API.

As a Java developer, it seems natural to start with the data model. Inevitably we end up with an abstract type to represent a message, which is accepted by the `sendMessage` method of [`MessagesClient`](https://github.com/Vonage/vonage-java-sdk/blob/main/src/main/java/com/vonage/client/messages/MessagesClient.java). Since the response we get back from sending a message is the same regardless of the message type and channel, we don't need to care about the message itself, we just forward it as a JSON payload and what we get back is a unique identifier of the message that was sent. With that in mind, our abstract [`MessageRequest`](https://github.com/Vonage/vonage-java-sdk/blob/main/src/main/java/com/vonage/client/messages/MessageRequest.java) factors out the fields common to all messages - namely: the type, channel, sender, recipient and an optional client reference string. Since the channels and message types are pre-defined, we represent them as enumerations.

There is a mixture of optional and mandatory parameters (for example, client reference is optional, but sender and recipient are mandatory). There are more parameters too, depending on the channel and message type combinations. Since Java does not have named arguments, we use the builder pattern to emulate this. You can read more about this [here](https://developer.vonage.com/blog/22/08/03/builder-pattern-with-inheritance-in-java).

We want to enforce that only valid combinations of message type and channel can be constructed, so we define this matrix in the [`Channel`](https://github.com/Vonage/vonage-java-sdk/blob/main/src/main/java/com/vonage/client/messages/Channel.java). We validate all arguments in constructors of `MessageRequest` and its subtypes. Herein lies what I believe is the main advantage of the design of the Java SDK's implementation of the Messages API: everything is ***correct by construction***.

What does this mean exactly? It means that in theory, it is difficult, ideally, impossible to construct an invalid message. I challenge anyone reading this to try to "misuse" the Messages API using the Java SDK. Let's try it together.

`MessageRequest` is abstract, so we have to use one of its concrete subtypes. All of these subtypes (for example, `MmsImageRequest`) already set the `Channel` and `MessageType`. Besides, even if we created our own subclass of `MessageRequest`, we still can't bypass the MessageType and Channel validation check since it's done in the constructor of `MessageRequest`. Furthermore, enums are implicitly final, so we can't add our own `MessageType` or `Channel` or override the validation behaviour. 

Furthermore, the constructors for all subclasses of `MessageRequest` are either protected (for abstract classes), or package-private (for concrete classes), and none of the concrete classes can be extended since they are `final` (same is true of the builders). So structurally, we cannot create a "bad" `MessageRequest` class.

Okay, let's try something else. If we can't structurally misuse `MessageRequest`, perhaps we can pass in invalid values to the builders. For example, could we send an empty text? Or what about passing in a nonsensical string in the `to` (recipient) field instead of a number? Or what if we omit mandatory parameters (such as the sender and recipient)? 

Oh, here's another one: all of the file-based message types (those that accept an image, audio, video etc.) have a URL field. Could we not pass in an invalid URL string? Or what about that Time-to-Live field in Viber messages? We could set a number that's out of the acceptable bounds since it's just an `int`, right? Oh, here's a niche one: in Messenger requests, both `Category` and `Tag` are optional. However, the API requires that if the `Category` has a value of `message-tag`, then the tag must be present. And what about...

Let's stop there. All of these cases have been considered, and none of them are possible. Yes, your code will compile if you try to pass in a malformed URL, an invalid number or neglect to set mandatory parameters on the builder. But at runtime, you will not be able to send the message. You won't even get as far as constructing the `MessageRequest` object! Why? Because every parameter is validated in the constructor. The moment you call `build()` on the Builder, the constructor is invoked, and if the parameters used to construct the message are missing or invalid, then you'll get an `IllegalArgumentException` thrown at you explaining the problem. This is what I meant when I said: "correct by construction". If you can construct a `MessageRequest`, then as far as we can tell, it's not obviously wrong, and you are clear to proceed to send it.

Of course, that doesn't mean that the message is completely valid or that you won't get back an error response from the server. You could, for example, try sending a text to a phone number that doesn't exist, yet is E164-compliant. This kind of validation is beyond the scope of the SDK and is performed by the backend service that the API communicates with. What the SDK guards against is essentially the 422 errors. So, what is this in contrast to? Where am I going with this? To answer that, let's look at the diametric opposite: cURL.

Why do I need the SDK when I can use the API directly? That's fine, here's an example of sending an image over Viber using cURL:

```sh
curl -X POST https://api.nexmo.com/v1/messages \
  -H 'Authorization: Bearer '$JWT\
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -d $'{
        "message_type": "image",
        "channel": "viber_service",
        "image": {
            "url": "https://example.com/image.jpg"
        },
        "to": "447700900000",
        "from": "9876543210",
        "viber_service": {
            "ttl": 600
        }
    }'
```

Whilst this is relatively easy to read, it is easy to get wrong when writing such a request manually. Not only do you need to remember the endpoint's URL, HTTP method, headers, authorisation type etc., but also the JSON format. You could miss the quotation marks (almost everything is a string), but watch out for that `ttl` - it's actually an integer, not an `int` wrapped in a string! You have to remember all of the fields and their exact naming - no IDE here is going to help you with auto-complete because you're just typing out text. 

Your terminal doesn't know anything about the API, so if you mistype something, there's no way to know until you send off the request and get a 422 response back. And remembering that schema, it's not really intuitive, is it? If you want to set the time-to-live, you have to remember to wrap it in a `viber_service` object. And don't forget the image URL has to be in an `image` object as well, oh and you can't include a caption with your image for Viber messages, but cURL won't stop you from trying. You can pass in whatever you want as the body. It doesn't even have to be valid JSON, let alone conform to the API schema!

It's fair to say then that to use the Messages API with cURL, you're going to need the API reference to hand and refer to it extensively to ensure that your requests are correct - not only the body but also the headers (authorisation method, for example). It's error-prone, and there's nothing to stop you from making mistakes or to help you formulate your intended message request. 

By contrast, if you want to use the Messages API through the Java SDK, you don't need the API reference at all. All of the public methods you need are documented, explaining what is required and what is optional, the usage pattern, examples etc. Even without reading the documentation, you can explore the API using nothing more than auto-completion provided by your IDE. For example, you won't try to set a caption on a Viber image because there is no way to do that in the Java SDK. So, as the user, you conclude that it's not supported.

What about discoverability? Well, all of the `MessageRequest` subclasses follow the `[Channel][MessageType]Request` naming scheme (for example, `SmsTextRequest`). You can even use your IDE to list all subclasses of `MessageRequest`. And there's only one way to construct requests: through the builder associated with each `MessageRequest` subclass. On top of that, the constructors for the builders are not public, so the only way to instantiate a builder is by calling the static `builder()` method on the class. This, combined with the package-private constructors, prevents you from misusing them.

A strongly typed language like Java prevents you from providing invalid values, not only at runtime but also at compile time. For instance, you may not know what the valid values are for `Category` in the optional `viber_service` object, but in the Java SDK, these are made obvious to you by the compiler / your IDE because [it's an enum](https://github.com/Vonage/vonage-java-sdk/blob/main/src/main/java/com/vonage/client/messages/viber/Category.java). 

And the whole thing about wrapping it in a `viber_service` object? You don't need to worry about that, you just set what you need on the builder, and the SDK will take care of it for you. In fact, the SDK takes care of all the serialisation and deserialisation. As a user, you don't know or need to care what serialisation format is being used behind the scenes - it's all taken care of. That's one of the main advantages of using an SDK instead of the API: it's *declarative* rather than *imperative*. You declare *what* you want to send, not *how* to send it.

Look at it this way: suppose that we decided all of our APIs are now going to use Avro or Protobuf (or, heaven forbid, plain old XML) instead of JSON. Those cURL scripts you had lying around are going to need a rewrite. Maybe if you're lucky and we kept the schema exactly the same, and you happen to be a RegEx guru, you could at least partially automate the migration. 

But what if we changed the schema? Sticking with the above example, what if we decided that the `url` parameter no longer needs to be wrapped in an object? Or we removed the `viber_service` container, which is used for `ttl` and `category` parameters? Or what if we renamed the `viber_service` message type to `viber` for consistency? If you're hand-rolling your requests with cURL, you need to be aware of all these changes. If you're using the Java SDK, you just need to upgrade to the next version - a single character change in your `pom.xml` or `build.gradle` file. No need to refactor anything - it's all taken care of behind the scenes.

Although I've been focusing on the Java SDK, it's worth noting that strong typing is not a prerequisite for validation. It just makes it easier to enforce by using the compiler as opposed to hand-coded logic. 

For example, the Python SDK's implementation of the Messages API is minuscule in size compared to the Java SDK ([it's a single file under 100 lines!](https://github.com/Vonage/vonage-python-sdk/blob/main/src/vonage/messages.py)), and [the Ruby SDK's](https://github.com/Vonage/vonage-ruby-sdk/tree/main/lib/vonage/messaging) is comparatively small as well. Both of these SDKs perform some basic validation on parameters. 

The same approach could have been used in the Java SDK: we could accept a Map construct with the parameters and validate them dynamically, but this is more error-prone since the compiler can't help us. The size of Java SDK's implementation is mostly due to Java being a verbose language; - for instance, it is similar to the [C# implementation](https://github.com/Vonage/vonage-dotnet-sdk/tree/main/Vonage/Messages) in principle, yet significantly more lines of code. 

It may be an interesting exercise for the reader to compare the initial Pull Requests for the Messages v1 implementation across the [Java](https://github.com/Vonage/vonage-java-sdk/pull/387/files), [Python](https://github.com/Vonage/vonage-python-sdk/pull/215/files), [Ruby](https://github.com/Vonage/vonage-ruby-sdk/pull/221/files) and [PHP](https://github.com/Vonage/vonage-php-sdk-core/pull/319/files) SDKs to assess this.

This strict approach to SDK design has its disadvantages, though. Let's suppose that at some point in the future, the API supports sending video over Viber. You can immediately take advantage of this with cURL, but an SDK which validates the message type and channel combinations will require an update. 

It's also a greater maintenance burden for the SDK maintainers when the API frequently changes because the validation logic, data types etc. all have to be updated to reflect these changes. But once things are settled and the API is stable (which is usually the case once our APIs transition from Beta to GA), this becomes less of an issue. Ultimately, we aim to provide the best possible user experience, and I hope this article has convinced you of the value that bespoke SDKs can add to REST APIs.

We always welcome community involvement. Please feel free to join us on the [Vonage Community Slack](https://developer.vonage.com/community/slack) or send us a message on [Twitter](https://twitter.com/VonageDev). If you have any suggestions for improvements, enhancements or you spot a bug, do not hesitate to raise an issue on [GitHub](https://github.com/Vonage/).
