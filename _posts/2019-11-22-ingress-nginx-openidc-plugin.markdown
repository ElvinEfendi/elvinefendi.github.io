---
layout: post
title: "Extending ingress-nginx to support OpenID Connect using its plugin system"
date: 2019-11-22 22:00
comments: true
tags: ingress-nginx, openidc, authentication, plugin, extensibility
---

I introduced [MVP version of Lua based plugin system for ingress-nginx ](https://github.com/kubernetes/ingress-nginx/pull/3807)
a while ago. Since then I've been meaning to write an article to demonstrate it but could not until now.

ingress-nginx does not support OpenID Connect out of box and there has been many requests for that by the community.
Therefore in this article I'm going to demonstrate ingress-nginx extensibility by showing how one can add OpenID Connect
functiontionality to it. This is going to be a tutorial like article so make sure you follow every step in order to acheive
a complete, working integration.

Let's first start by deploying latest released version of ingress-nginx locally using minikube:

```
mkdir -p $GOPATH/src/k8s.io
cd $GOPATH/src/k8s.io
git clone https://github.com/kubernetes/ingress-nginx.git && cd ingress-nginx
```
```
git checkout nginx-0.26.1
```
```
make dev-env
```

Note that you don't have to compile it from source.

After you should see running ingress-nginx pods when you execute

```
kubectl get po -n ingress-nginx --context minikube
```

Next step is to deploy an example app.

```
kubectl create ns example-app

kubectl create -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/docs/examples/http-svc.yaml -n example-app --context minikube
```

Now that we have service and deployment, let's create the corresponding ingress:

```
echo '
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example-app
spec:
  rules:
  - host: example.com
    http:
      paths:
      - backend:
          serviceName: http-svc
          servicePort: 80
        path: /
' | kubectl create -n example-app --context minikube -f -
```

Let's also make sure `example.com` resolves to the local minikube endpoint:

```
echo "$(minikube ip) example.com" | sudo tee -a /etc/hosts
```

Also this is a good place to note down the port ingress-nginx is running behind:

```
> ingress-nginx ((HEAD detached at nginx-0.26.1))$ minikube service list | grep ingress-nginx
| ingress-nginx | ingress-nginx | http://192.168.99.108:31672    |
```

For me the HTTP port is `31672`, and therefore the :endpoint I'll be using is `http://example.com:31672`.

At this point you should be able to access http://example.com:31672 in your browser.

Now it is time to add OpenID Connect based authentication for our example app. To make things easier I've created
a repository with all the necessary things in https://github.com/ElvinEfendi/ingress-nginx-openidc. However since that's
a barebone I will modify it to produce a fully working demonstration here.

*Before we continue let me introduce you to ingress-nginx's plugin system first.* An ingress-nginx plugin is a set of Lua modules
placed in `/etc/nginx/plugins/<plugin name>/`. Every plugin has to have a `main.lua` file in its root directory. This file
defines the Nginx phases you want to run your custom logic in. For example if you want to run a custom logic in `rewrite` phase
you need to declare a global function called `rewrite`, for example: https://github.com/ElvinEfendi/ingress-nginx-openidc/blob/195b13476039bb9dc39ad8b418a584d1753dd659/rootfs/etc/nginx/lua/plugins/openidc/main.lua#L17.
Currently following phases are supported: `init`, `rewrite`, `header`, `log`. Once the plugin is ready the final step is to enable
it in `nginx.tmpl`, for example: https://github.com/ElvinEfendi/ingress-nginx-openidc/blob/195b13476039bb9dc39ad8b418a584d1753dd659/rootfs/etc/nginx/template/nginx.tmpl#L121. `plugins.init({})` receives array of plugin names as an argument and then takes care of safely loading them. However note that this plugin system is not meant for untrusted code as they are not run in a sandbox.
In case your plugin requires third party lua libraries, you can install them in the custom `Dockerfile`, for example: https://github.com/ElvinEfendi/ingress-nginx-openidc/blob/195b13476039bb9dc39ad8b418a584d1753dd659/rootfs/Dockerfile#L5. Sometimes a plugin might also require additional Nginx configuration, this can be done in the custom `nginx.tmpl`, here's an example: https://github.com/ElvinEfendi/ingress-nginx-openidc/blob/195b13476039bb9dc39ad8b418a584d1753dd659/rootfs/etc/nginx/template/nginx.tmpl#L53.

Now let's adjust that plugin for this demonstration and show full flow. To do so we will need client id and client secret. I'm using
Google as an authentication provider in this tutorial. https://developers.google.com/identity/protocols/OpenIDConnect shows how you can
obtain them. When creating OAuth client ID in Google Console make sure you choose "Web Application" as an application type and
set `http://example.com:31672` in "Authorized JavaScript origins" and `http://example.com:31672/redirect_uri` in "Authorized redirect URIs". Also you will need to set `example.com` in "Authorized Domains" in "OAuth consent screen" page. If you have different port for
ingress-nginx in minikube, adjust the entries accordingly.

Next we will create a secret with the credentials. First make sure you have files called `client_id` and `client_secret` with the
corresponding credentials you obtained from Google. After the create the secret:

```
kubectl create secret generic openidc-credentials --from-file client_id --from-file client_secret -n ingress-nginx --context minikube
```

Make sure the files does not have newline character.

Next make the following change to the barebone plugin:

```
diff --git a/rootfs/etc/nginx/lua/plugins/openidc/main.lua b/rootfs/etc/nginx/lua/plugins/openidc/main.lua
index 56dcb23..c1580ca 100644
--- a/rootfs/etc/nginx/lua/plugins/openidc/main.lua
+++ b/rootfs/etc/nginx/lua/plugins/openidc/main.lua
@@ -8,7 +8,7 @@ local CLIENT_SECRET = os.getenv("OPENIDC_CLIENT_SECRET")
 local opts = {
   -- For full list of options see https://github.com/zmartzone/lua-resty-openidc

-  redirect_uri = "https://example.com/redirect_uri",
+  redirect_uri = "http://example.com:31672/redirect_uri",
   discovery = "https://accounts.google.com/.well-known/openid-configuration",
   client_id = CLIENT_ID,
   client_secret = CLIENT_SECRET,
```

Since we are using HTTP endpoint with a specific port, that change is necessary.

Now switch to minikube's docker

```
> ingress-nginx-openidc (master)$ eval $(minikube docker-env)
```

and build the custom ingress-nginx image:

```
> ingress-nginx-openidc (master)$ docker build -t ingress-nginx-openidc rootfs/
```

Since our plugin requires the credentials to be passed as environment variables we need to define them in ingress-nginx deployment manifest. We also need to change the image to our custom image `ingress-nginx-openidc`. By editing the ingress-nginx deployment, make sure
you include following in the `env` key:

```
env:
- name: OPENIDC_CLIENT_ID
  valueFrom:
    secretKeyRef:
      name: openidc-credentials
      key: client_id
- name: OPENIDC_CLIENT_SECRET
  valueFrom:
    secretKeyRef:
      name: openidc-credentials
      key: client_secret
```

and change the image to `ingress-nginx-openidc`. Now when you try to access http://example.com:31672/ it should redirect you to Google sign in page. Here's how it looks like for me after I sign in:

![ingress-openidc](../assets/ingress-openidc.png)
