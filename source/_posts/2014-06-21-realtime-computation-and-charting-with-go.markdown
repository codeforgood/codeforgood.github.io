---
layout: post
title: "Realtime Computation and Charting with Go and d3"
date: 2014-06-21 15:23:04 -0400
comments: true
author: Sadish Ravi
categories: architecture, go, d3, realtime, computation, charting
---

Realtime web systems are always fun and challenging to build. To try my hands on this, I worked with a team and built a  realtime student assessment metrics dashboard during a 2 days hackathon. We built this using go, reveal, redis and d3.

#### What did we build?

At my current work, I have worked on building student assessment data processing systems. These are mostly batch based systems and the delay associated with them usually works out fine for our clients. But there are still unexplored use cases for realtime computations in education.

The system that we built listens to a realtime feed (student assessment information), does various computations/aggregations in parallel and pushes the result to a result queue. The web socket application listens to this computation result feed and updates the dashboard in realtime

#### Parts of the system

##### Realtime feed
we simulated a realtime feed of student assessment information using go. A goroutine reads a csv file and sends data to a go source feed channel at regular intervals
##### Computations
Bunch of goroutines listens to the above channel and performs some computation on the record in parallel. These computaitons are usually dependent on the computation result till now. We can do this either by storing the result in database temporarily or using a closure to keep track of result till now. Some calculations we implemented were
    - Average score by school
    - Top performing schools in a district
    - Top performers in a grade across schools in a district
    - ....
Each of the above calculation was implemented as a goroutine and they push the result to a result channel as string. These routines listens to the source feed channel and runs as long the source closes it channel
##### Aggregator
Aggregator can be seen as the main routine which initiated all the above computation routines and listens for results on the output channel for each of the routines. The aggregator does 2 things
    - publishes the current result from a computation to its own redis queue
    - Appends the result to a snapshot result list in redis.This is needed for initial state of the chart up on loading
##### Socket server
The socket server was built using go and reveal framework. Its a very simple endpoint, which subscribes to each of the above redis computation queue and emits the data up the stack to browser as a message tagged with the chart on which it needs to be shown
##### D3 charts
The dashboard ui is built using d3 charts and each of the chart listens to a particular message on the socket connection and redraws themselves with new data

#### Putting together

{% img left /images/Dash/dash-arch.jpg 'Dash Architecture' 'Dash' %}
