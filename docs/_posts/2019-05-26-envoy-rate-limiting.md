---
layout: post
title:  "Envoy Rate Limiting"
date:   2019-05-26 18:32:21 -0500
categories: envoy
---

[Envoy](https://www.envoyproxy.io) offers global rate limiting as a feature. However, Envoy isn't able to do this by itself. Ut needs to work on concert with a backing rate limit service for storing and tracking request information across a number of different processes on different instances.

Envoy can handle rate limiting based on each new connection, calling
the rate limit service for each new connection. Or for HTTP based based rate limiting calling the rate limit service for each new request.
