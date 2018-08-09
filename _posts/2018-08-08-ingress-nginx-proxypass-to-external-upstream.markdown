---
layout: post
title: "proxying to an external endpoint using ingress-nginx"
date: 2018-08-08 10:30
comments: true
tags: ingress, nginx, kubernetes, IngressNGINX, externalname, rewrite, server-snippet
---

The article demonstrates how to proxy to an external domain or a different app in [ingress-nginx](https://github.com/kubernetes/ingress-nginx/) using a service of type `ExternalName` and the following annotations:

 - <https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#rewrite>
 - <https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#custom-nginx-upstream-vhost>

and optionally:

 - `nginx.ingress.kubernetes.io/backend-protocol`
 - <https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#server-snippet>

Assume you have an app running at `existing.com`. And you'd like to publish one of the pages in that website as a different website. Let's say the page is at `existing.com/separate-page`. Now you want to publish that page as a completely different website at `new.com`. First you need to create a Service resource of type `ExternalName` as following:

```
kind: Service
metadata:
  name: myservice
spec:
  externalName: plus-website.shopifycloud.com
  type: ExternalName
```

Then edit the Ingress resource for the same app and add the following annotations:
```
ingress.kubernetes.io/rewrite-target: "/separate-page"
ingress.kubernetes.io/upstream-vhost: "existing.com"
```

So your Ingress should look something like
```
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: "/commerceplus"
    nginx.ingress.kubernetes.io/upstream-vhost: "existing.com"
  name: google-ingress
spec:
  rules:
  - host: new.com
    http:
      paths:
      - backend:
          serviceName: myservice
          servicePort: 80
        path: /
```

If you want to proxy over `HTTPS` then in the Ingress definition above you should set `servicePort` to `443` (or whatever the upstream listening for HTTPS requests on), `nginx.ingress.kubernetes.io/backend-protocol` to `https` and also include the following configuratoin
```
nginx.ingress.kubernetes.io/server-snippet: |
    proxy_ssl_name existing.com;
    proxy_ssl_server_name on;
```
if the upstream requires SNI.

Now whenever you access `new.com` Nginx will serve content from `existing.com/separate-page`.
