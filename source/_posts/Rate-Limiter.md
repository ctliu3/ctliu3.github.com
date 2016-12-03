---
title: Rate Limiter
date: 2016-12-03 10:52:19
categories: Engineering
tags: [rate-limiter]
---


I dig into rate limiter these days. Rate limiter is a tool that limits the number of request in a given short period. Any service only can handle limited requests with limited resources (cpu, memory, etc). Rate limiter prevents service to reach a high load that beyond its load capacity.

<!--more-->

Generally, rate limiter can be implemented using leaky bucket or token bucket algorithms. The main difference between them is the rate accepting a request. The word "bucket" is really imagery word that can help you catch the idea of the algorithms.

For leaky bucket algorithm, the time gap between any two adjacent requests should be greater than a fixed value. Therefore, if there comes huge requests in a short period, the requests will through the service one by one, no burstiness happens. However, the burstiness occurs in token bucket algorithm. Before passing the limiter, request should check if there is enough tokens left. If there is, then the request is accepted. The tokens will be refilled in a given period. If many requests get the tokens at almost the same time, then burstiness takes place.

For a full-featured rate limiter, following points should be considered:

- Multiple request at a time. The requests are packed into batch, they should be handled simultaneously.
- Multiple buckets. Each bucket is used for a namespace, such as user id or host.
- Heterogeneous request type. You need different limiter to handle different type of requests.
- Dynamic configuration. Reconfigure the limiter without restarting it.
- Make it distributed. Multiple same service use a central limiter. But this will make the limiter a little bit heavy and I don't recommend this in some degree.
- etc...

In [tap](https://github.com/ctliu3/tap), which is a project in github, I implement the above two algorithms of rate limiter using golang. Also, I also extend the rate limiter to a distributed version, backend with redis. This project is a good start to understand rate limiter, but it's still an experimental project, be careful if you want to use it in production environment. I don't suppose to get into details about the tap because it's not completed finished, you can take a glance on the codes.