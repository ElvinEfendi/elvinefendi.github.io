---
layout: post
title:  "distributed calendar app in Java and Ruby"
date:   2014-05-06 11:00
comments: true
tags: [distributed systems, java, ruby, ricart agrawala, mutual exclusion, token ring, xml rpc]
---

Finally I have got some time and enough passion to write a blog post about one of my courseworks I've done during last semester. The task was to
implement a simple calendar network, where people can join network, start new network, quit network and CRUD appointments. The requirement
was that all appointments should be syncronized among the machines in the node and there should not be any inconsistency and collision.
Moreover it required us to write the same project in two different programming languages and user should be able to join to the network
using calendar app implemented in different languages. I've chosen Java and Ruby and used [XML RPC](http://en.wikipedia.org/wiki/XML-RPC) 
to connect the nodes in the network. To avoid collisions such as editing an appointment at the same time I've implemented Token Ring and
Ricart agrawala [mutual exclusion](http://en.wikipedia.org/wiki/Mutual_exclusion) algorithms. 
Before running the application it is possible to set the mutual exclusion algorithm.
I've published the code for both languages in github: [Ruby](https://github.com/ElvinEfendi/distributed-calendar-app-ruby) and
[Java](https://github.com/ElvinEfendi/distributed-calendar-app-java)

Further instructions on how to run the program can be found in the repository.
