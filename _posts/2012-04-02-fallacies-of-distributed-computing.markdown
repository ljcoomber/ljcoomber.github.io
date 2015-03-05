---
layout: post
title:  "The Fallacies of Distributed Computing"
date:   2012-04-02 18:49:47
categories: misc
---

This post first appeared on the [LShift web site](http://www.lshift.net/blog/2012/04/02/the-fallacies-of-distributed-computing/)

I recently helped a client take ownership of an application stack produced by a third party. There were various components to this, but the most interesting aspect allowed users to interact with each other by manipulating images in a browser. This was achieved using a small and simple [NodeJS](http://nodejs.org/) application to send events between the browsers.

We were aware of a few severe problems shortly before our involvement, and during the handover meeting I asked the developer how he had fixed them. He replied that if two or more users tried to manipulate the same object at the same time, it would cause the browser code to crash as multiple conflicting change events would be sent. He therefore added a new event type that was sent out when an action started, e.g., on mouse down. This indicated that all the other browsers should lock their copies of the object in question until the action was finished.

I gently suggested that this might not work well over a network, to which he responded that in testing the lock events took less than 10ms to arrive, so the odds of two people trying to do the same thing within that period were so small as to not be worth worrying about.

Again, I very gently suggested that the benchmark was unlikely to hold across the Internet. He told me that he had tested it on several networks including over the public Internet with someone in the US and it was still good. I gave up at this point, and just noted to the client that this was something that would need attention fairly promptly.

This assertion that there was a latency of 10ms between the UK and the US is clearly false. Even if you could set-up a perfect vacuum between London and New York, it would still take (19ms for light to travel that distance)[19ms for light to travel that distance]. However, the numbers are irrelevant – there is latency and it must be considered when trying to build a reliable distributed system regardless of magnitude. This and seven other issues were famously covered in the “Eight Fallacies of Distributed Computing”, assembled by Peter Deutsch, James Gosling and others.

Anyone building a distributed application should have these fallacies in mind, and [this paper](http://www.rgoarchitects.com/Files/fallacies.pdf) has a good explanation of them with a very readable commentary. NodeJS made this shared browser experience extremely effective and enticingly simple to construct, but even NodeJS apps are not isolated from the effects of the infrastructure they utilise. In fact, until TCP over [entangled photon pairs](http://www.physorg.com/news193551675.html) hits the shelves, these fallacies will be relevant regardless of your framework.
