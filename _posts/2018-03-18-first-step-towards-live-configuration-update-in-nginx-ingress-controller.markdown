---
layout: post
title: "first step towards live configuration update in nginx ingress controller"
date: 2018-03-18 12:50
comments: true
tags: ingress, nginx, kubernetes
---

**tldr;** A new experimental feature introduced at  [https://github.com/kubernetes/ingress-nginx/pull/2174](https://github.com/kubernetes/ingress-nginx/pull/2174) allows us to skip Nginx reloads on backend changes/deployment.

Nginx Ingress Controller is a Kubernetes controller that watches updates on ingress resources of certain class and configures Nginx
accordingly. Everytime there's an update it generates a new corresponding Nginx configuration file and reloads Nginx workers
so that they pick the new configuration. Even though Nginx handles reloads gracefully it is still desirable to avoid them because
they result in increased memory consumption(for certain period of time while the old workers are around), reset of load balancing state and reset of the keepalive connections which lead to increased latency for end users. In a Kubernetes cluster everytime an application is deployed
the IP addresses of its pods change. That means the controller has to create a new configuration with the list of updated upstream peers and
reload Nginx workers. Now imagine you have 50 applications running behind the same Nginx Ingress Controller and each of them gets deployed
10 times per 8 hours(a work day) on average. Given the ideal distribution of those changes we would have more than one(`1.04166667`)
reload per hour. This would mean your users would experience degraded performance every next hour. And this gets worse and worse with
the number of applications.

In order to avoid this shortcoming Nginx Ingress Controller now supports a new experimental feature where when enabled it skips reloads for
backend changes(pod IP changes are one of them). This was achieved by using `balancer_by_lua` directive offered by [lua-nginx-module](https://github.com/openresty/lua-nginx-module). When this feature is enabled the controller will skip reload and post the new list of backends
to a Lua endpoint and Lua will take care of load balancing. Currently only Round Robin load balancing algorithm is supported.
The feature was introduced at [https://github.com/kubernetes/ingress-nginx/pull/2174](https://github.com/kubernetes/ingress-nginx/pull/2174)
and can be enabled by starting `nginx-ingress-controller` with `--enable-dynamic-configuration`. It's going to be included in the next 
release of Nginx Ingress Controller but in the meantime you can use `index.docker.io/elvinefendi/nginx-ingress-controller:0.12.0`
to try out this new experimental feature. Refer to [this](https://github.com/kubernetes/ingress-nginx/blob/master/deploy/README.md)
instructions for how to install Nginx Ingress Controller in your cluster.

The next immediate steps are going to be adding more load balancing algorithms to match the default Nginx LB features and skipping
reloads when certificates change by using `certificate_by_lua` directive.
