---
layout: post
title:  "Apache Camel and RabbitMQ"
date:   2011-02-04 18:49:47
categories: misc
---

This post first appeared on the [LShift web site](http://www.lshift.net/blog/2011/02/04/apache-camel-and-rabbitmq/).

I’m evaluating [Apache Camel](http://camel.apache.org/) for use on a client project, but we need to back it on to [RabbitMQ](http://www.rabbitmq.com/). The AMQP component that comes with Camel is based on the Qpid 0.5.0 client which [does not work too well with Rabbit](http://www.rabbitmq.com/interoperability.html), so this seemed a good excuse to experiment with custom Camel components.

There’s a first pass on [GitHub](https://github.com/lshift/camel-rabbitmq) for anyone who wants to play. Note the phrase “first pass” and that the Limitations section of the README file is longer than the Usage one.
