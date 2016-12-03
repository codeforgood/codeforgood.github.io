---
layout: post
title: "Iot bell: a doorbell using Aws iot button"
date: 2016-12-02 20:51:11 -0800
comments: true
author: Sadish Ravi
categories: [architecture, iot, doorbell, aws, lambda, slack]
---

A Doorbell for 20$ that slacks instead of ringing. This idea was given birth due to many reasons

    - Our office floor was missing a doorbell for a long time
    - People with out key fob had to tail gate others after a long wait
    - We wanted to put to a use a spare aws iot button


### How it works?

{% img left /images/doorbell/doorbell-arch-diag.png 'Doorbell Architecture' 'doorbell' %}


### Deep dive

#### iot Button

[AWS iot button](https://aws.amazon.com/iotbutton/) is a simple device that triggers an  event upon clicking. Pretty much any piece of code/action can be executed by handling this event. [purchase aws iot button here](https://www.amazon.com/dp/B01C7WE5WM).

#### Slack webhook

We will be using a slack webhook url to post a message to slack. A webhook url can be configured and obtained from slack settings.

#### Lambda function
This a very simple python script that gets executed upon an iot event and posts a message to a slack channel.

While creating a lambda function, select the trigger to be AWS IoT and enter the button device serial number to generate certificate and key. This will be used later to setup the button.

The lambda function itself is pretty straightforward

{% codeblock iotbell function - iotbell.py %}

HOOK_URL = '<webhook url from slack>'
SLACK_CHANNEL = '<slackchannel to post>'

def lambda_handler(event, context):
    logger.info("Event: " + str(event))
    dsn = event['serialNumber']
    logger.info("Received trigger from device with dsn: " + dsn)

    slack_message = {
        'channel': SLACK_CHANNEL,
        'parse': 'full',
        'text': "%s" % ('@here DING DONG!!! Open door please. Triggered from 6th floor door bell.')
    }

    req = Request(HOOK_URL, json.dumps(slack_message))
    try:
        response = urlopen(req)
        response.read()
        logger.info("Message posted to %s", slack_message['channel'])
    except HTTPError as e:
        logger.error("Request failed: %d %s", e.code, e.reason)
    except URLError as e:
        logger.error("Server connection failed: %s", e.reason)
{% endcodeblock %}

This is pretty much the gist of the function. We also encrypted the HOOK_URL using aws keys management service, but thats optional. AWS lambda iot sample function has instructions on how to do this.

#### Button setup
Once we have the certificate and private key, button can be setup by following these [instructions](http://docs.aws.amazon.com/iot/latest/developerguide/configure-iot.html).


#### Slack message

{% img left /images/doorbell/slack-message.png 'Slack message' 'slack-message' %}
