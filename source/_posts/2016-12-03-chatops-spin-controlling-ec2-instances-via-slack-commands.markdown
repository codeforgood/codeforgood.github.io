---
layout: post
title: "ChatOps spin: Controlling EC2 instances via slack commands"
date: 2016-12-03 17:01:39 -0800
comments: true
categories: [aws, chatops, ec2, lambda, apigateway, slack]
---
[ChatOps](https://www.youtube.com/watch?v=NST3u-GjjFw) term mostly credited to Github is a way of integrating tools, operations, scripts and humans with a chat interface (Slack, HipChat etc...).

At work we run some ec2 hosted tools for other non engineering teams. We don't have the need to run it 24*7, so it would be great to give the ability to users to spin up the instance and bring it down when needed to save costs. For some time engineering was doing this upon request, but this is something that could be easily automated. I was interested to see how this could be done via slack.

After reading few articles writing bots using lambda, I decided to go with that.

### How it works?

{% img left /images/chatops/ec2-controller.png 'chatops Architecture' 'chatops' %}


### Deep dive

I decided to build this using lambda, rather than any other open source bot frameworks as we were trying to adopt usage of lambda internally for other purposes, due to its tight integration with aws services and no need to host anything (serverless).

#### Slack command

A Slack command can be created and tied to a url to which the data will be posted.

#### API gateway

In order to execute a lambda function upon a slack command, we need a API to which slack can send a get/post request. AWS makes it easier to create a API using API gateway that will invoke a particular lambda function when called.

I am not going to go in details on how to create a API gateway and connect to a lambda function as these are explained in depth in [AWS docs](http://docs.aws.amazon.com/apigateway/latest/developerguide/getting-started.html)

#### lambda function

Although I started with a single lambda function that was responding to 3 different slack requests
    - /ec2 status (GET call to know the status of the ec2 instance)
    - /ec2 start  (POST call to start the ec2 instance if not running)
    - /ec2 stop   (POST call to stop the ec2 instance if running)

soon I realized its not going to work as slack terminates outbound requests if it gets no response within 3 seconds. One solution was to split the response and the actual work. Getting a status of the instance could be done with in 3 seconds, but starting and stopping the instances are synchronous calls and could easily take more than 3 seconds.

The first lambda function I wrote was a reception function whose only work was to ack the slack request from API gateway and respond with something meaningful. The reception function will in turn trigger a lambda worker function to do the actual work (Querying status, starting or stopping an ec2 instance)

{% codeblock reception function - receptor.py %}

import json
import boto3

SLACK_TOKEN = '<TOKEN>' # to ensure the request is coming from slack
SNS_TOPIC_ARN = '<ARN>' # Sns topic to which notification will be sent

def lambda_handler(event, context):
    if event.get('token', None) != SLACK_TOKEN:
        return 'Unauthorized request. Check slack token passed'

    if event.get('command', None)  != '/EC2':
        return 'Invalid command. Check slack command passed'

    request = event.get('text', None)
    if request not in ['status', 'start', 'stop']:
        return 'Unknown request. Check slack command passed'

    sns_client = boto3.client('sns')
    sns_client.publish(
        TopicArn=SNS_TOPIC_ARN,
        Message=json.dumps({'default': json.dumps(event)}),
        MessageStructure='json'
    )
    return 'Will get back to you on that.'

{% endcodeblock %}

The function verifies if the request is from slack and checks the slack command that was issued. After checking the request, it sends a sns notification to a topic with the entire event data as message. The function gets back to the original request with a placeholder message. SNS is one easy way to chain lambda functions.

A downstream worker lambda function was written which gets triggered when new message is sent to the above topic. The function will receive the entire message sent to sns topic. This message will include the original request that was received by the receptor function.

{% codeblock worker function - worker.py %}

import boto3
import json
import urllib2

region = '<>'
instance_id = '<>'
SLACK_TOKEN = '<>'

def post_to_slack_webhook(response_url, text):
    try:
        req = urllib2.Request(response_url)
        req.add_header("Content-Type", "application/json")
        urllib2.urlopen(req, data = json.dumps({'text': text}))
    except urllib2.HTTPError as e:
        pass

# handler for sns notification triggered from receptor lambda function
def lambda_handler(event, context):
    message = json.loads(event['Records'][0]['Sns']['Message'])

    command = message.get('command', None)
    action = message.get('text', None)
    response_url = message.get('response_url', None)

    ec2 = boto3.resource('ec2')
    instance = ec2.Instance(instance_id)

    if action == 'status':
        if instance.state['Code'] == 80:
            text =  "Instance is currently stopped. To start issue /ec2 start"
        elif instance.state['Code'] == 64:
            text =  "Instance is currently stopping. Check back in few minutes"
        elif instance.state['Code'] == 16:
            text =  "Instance is currently running. To stop issue /ec2 stop"
        elif instance.state['Code'] == 0:
            text =  "Instance is currently starting. Check back in few minutes"
        else:
            text = "Instance is in unexpected state"
        post_to_slack_webhook(response_url, text)
    elif action == 'start':
        if instance.state['Code'] == 16:
            post_to_slack_webhook(response_url, 'Instance already running')
        elif instance.state['Code'] == 0:
            post_to_slack_webhook(response_url, 'Instance just booting up')
        else:
            instance.start()
            instance.wait_until_running()
            ec2.meta.client.associate_address(InstanceId=instance_id, PublicIp='107.20.248.29')
            post_to_slack_webhook(response_url, 'Instance now running')
    elif action == 'stop':
        if instance.state['Code'] == 80:
            post_to_slack_webhook(response_url, 'Instance already stopped')
        elif instance.state['Code'] == 64:
            post_to_slack_webhook(response_url, 'Instance just shutting down')
        else:
            instance.stop()
            instance.wait_until_stopped()
            post_to_slack_webhook(response_url, 'Instance stopped')
    else:
        post_to_slack_webhook(response_url, 'Invalid action requested!')
{% endcodeblock %}

The slack request also includes a response_url to which one can post a delayed response. The worker function after parsing the message, takes the appropriate action and responds back to the response_url with a appropriate text.

### Slack interaction

{% img left /images/chatops/ec2-status.png 'ec2 status' 'status' %}

### Credits
  - [So, What is ChatOps?](https://www.pagerduty.com/blog/what-is-chatops/)
  - [ChatOps at GitHub by Jesse Newland](https://www.youtube.com/watch?v=NST3u-GjjFw)
  - [ChatOps at GitHub](https://speakerdeck.com/jnewland/chatops-at-github)
