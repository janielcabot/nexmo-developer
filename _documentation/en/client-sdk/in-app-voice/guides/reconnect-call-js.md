---
title: Reconnect a call or media with JavaScript
description: How to reconnect a call or conversation's media with the JavaScript Client SDK.
navigation_weight: 8
---

# Reconnect a Call or Conversation's Media

## Overview

This guide covers how to reconnect to a call or a conversation's media in your JavaScript Vonage Client application.

### Automatically Reconnect

The JavaScript Client SDK will automatically reconnect to a [call](https://developer.vonage.com/sdk/stitch/javascript/Application.html#reconnectCall) or a conversation's [media](https://developer.vonage.com/sdk/stitch/javascript/Media.html#enable) if a connection goes out for a couple of seconds without you needing to do anything. 

The following is for the situations where a browser window/tab is accidentally closed/refreshed or the user wants to switch audio inputs and are still within the 20-second time limit to reconnect.

### Considerations

There are a few things that need to be in place for the reconnection to happen successfully.

1. The application must create a new JWT for the reconnecting user with their unique username in order to instantiate the [Application Object](https://developer.vonage.com/sdk/stitch/javascript/Application.html) which has the `reconnectCall` and `getConversation` methods.
2. The Conversation Id and RTC (or Leg) Id will also be needed to reconnect the user.

This means that these values (username, Conversation Id, and RTC Id ) will need to be stored in the browser so they can be recovered when the application reloads. Both [localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage) and [sessionStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage) can be used to store the data. The one you choose will depend on your use case.

There is an optional `mediaParams` parameter that can be added to modify the MediaStream object and do things like changing the audio input device. Visit the [documentation](https://developer.vonage.com/sdk/stitch/javascript/Media.html#enable) to view all the arguments.

### Reconnect a Call

```tabbed_content
source: '_tutorials_tabbed_content/client-sdk/guides/reconnect-call/javascript/call'
frameless: false
```
A sample application can be found in this GitHub [repo](https://github.com/Vonage-Community/sample-clientsdk-js-reconnect_call).

>Note: When reconnected, you will automatically join the call. No need to `answer()` on the Call Object. 

### Reconnect a Conversation's media

```tabbed_content
source: '_tutorials_tabbed_content/client-sdk/guides/reconnect-call/javascript/conversation'
frameless: false
```
