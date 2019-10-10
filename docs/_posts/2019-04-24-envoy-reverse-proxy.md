---
layout: post
title:  "Envoy Reverse Proxy"
date:   2019-04-24 18:32:21 -0500
categories: envoy
---

[Envoy](https://www.envoyproxy.io) is a modern proxy designed for service oriented architectures. Envoy was initially developed at Lyft to simplify communication between their growing fleet of services created by a number of different development teams.

Since being released as open source, several large companies including Amazon and Google have invested in the development of Envoy. These companies have even created products using Envoy as the core of the product offering. This includes an Istio offering on GCP, and AWS AppMesh.

Initially I didn't really **get** Envoy. I asked myself, "Why would I use Envoy over [Nginx](https://www.nginx.com/)?” I looked at some of the documentation and the configuration of Envoy looked a lot more complex than __what I had used__ with Nginx.

There are good reasons why the Envoy configuration is complex. The problem domain Envoy is targeting is very complex. Envoy is targeting several large concerns: TLS, routing, traffic shifting, traffic mirroring, HTTP/2, gRPC, health checking, load balancing, retries, rate limiting, circuit breaking, observability, and more.

Envoy supports both static configuration such as a YAML file loaded when the process starts, dynamic configuration based on watching files on the filesystem, or polling an API to "discover" the routing rules for Envoy to use. The dynamic configuration is especially interesting when using Envoy as part of a service mesh to enable discovery of other services in the mesh without redeployment or configuration by hand.

Today we’ll look at some static configuration to use Envoy in a simple reverse proxy use case with docker and docker compose.

### Follow along

I would expect you to be using macOS, linux, or Windows WSL to follow along. Feel free to work with whichever tools make sense for you.

Make sure you have installed [Docker Desktop](https://docs.docker.com/install/) or the docker CLI and that docker is running. You will also need [`docker-compose`](https://docs.docker.com/compose/install/https://docs.docker.com/compose/install/) if you would like to follow along with the commands. On macOS these can be installed with the following.

```bash
brew cask install docker       # install docker desktop
open /Applications/Docker.app  # run docker desktop
brew install docker-compose    # install docker-compose
```

Create an initial directory for our files.

```bash
mkdir envoy-reverse-proxy
cd envoy-reverse-proxy
```

These 2 commands will pull the 2 docker images we will run from [dockerhub](https://hub.docker.com/). The first image is Envoy itself and the second is a python webservice that you can see running at [httpbin.org](http://httpbin.org), but we will be running it locally for testing purposes.

```bash
docker pull envoyproxy/envoy
docker pull kennethreitz/httpbin
```

You can confirm that the images are on your machine by running `docker images`. This will display the list of container images that are local to your machine. Your information may be different from mine but, as long as `envoyproxy/envoy` and `kennethreitz/httpbin` are listed, it should work out ok.

```bash
docker images
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
envoyproxy/envoy                           latest              20b550751ccf        4 weeks ago         164MB
kennethreitz/httpbin                       latest              b138b9264903        6 months ago        534MB
```

Create a file for our Envoy configuration.

```bash
touch envoy-reverse-proxy.yml
```

Copy the following into `envoy-reverse-proxy.yml`.

```yaml
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address: { address: 0.0.0.0, port_value: 9901 }
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 10000 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: httpbin }
          http_filters:
          - name: envoy.router
  clusters:
  - name: httpbin
    connect_timeout: 0.25s
    type: LOGICAL_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: httpbin
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: httpbin
                port_value: 80
```

Create a file for `docker-compose` configuration.

```
touch docker-compose.yml
```

Copy the following into `docker-compose.yml`

```yaml
version: '3'
services:
  httpbin:
    image: "kennethreitz/httpbin"
    ports:
     - "80"
  envoy:
    image: "envoyproxy/envoy"
    ports:
      - "9901:9901"
      - "10000:10000"
    volumes:
      - "./envoy-reverse-proxy.yml:/etc/envoy/envoy.yaml"
```

Run `docker-compose up` in your terminal and browse to [`localhost:9901`](http://localhost:9901) to see the configured Envoy admin or [`localhost:10000`](http://localhost:10000) to see our local instance of `httpbin`.

<div style="font-size: 40px; display: flex;">
  <div style="text-align: right; flex: 1; padding-right: 5px;">🎉 🎉 🎉</div>
  <div style="text-align: right; flex: 1; padding-right: 5px; transform: scale(-1, 1);">🎉 🎉 🎉</div>
</div>

### How did that work

Let's break down the Envoy configuration to see what's going on here.

![Envoy Reverse Proxy Configuration Annotated](/assets/envoy-reverse-proxy.png)

The graphic shows that the admin interface is started on port 9901 and another listener is started on port 10000. The listener is using the `envoy.http_connection_manager` TCP filter, that TCP filter uses the `envoy.router` HTTP filter, and it has route configuration to route all domains and all paths to the cluster named `httpbin`. Lower in the configuration the `httpbin` cluster is defined with a load assignment to route traffic to the `httpbin` DNS name on port 80.

![Envoy Reverse Proxy docker-compose configuration Annotated](/assets/envoy-reverse-proxy-docker-compose.png)

I hope you made it this far and learned something about Envoy! If you enjoyed this, signup to be notified when more content like this is posted.
