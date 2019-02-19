---
layout: post
title: "canary deployment with ingress-nginx"
date: 2018-11-25 10:30
comments: true
tags: ingress, nginx, kubernetes, IngressNGINX, canary, deployment, Blue/Green, A/B testing
---

_Note_: Make sure you're using ingress-nginx not older than version `0.22.0`. The initial implementation of canary feature had serious flaws that got fixed in https://github.com/kubernetes/ingress-nginx/releases/tag/nginx-0.22.0.

Canary and Blue/Green deployment, A/B testing are all known techniques to safely rollout a new version of service. Most of the times it's required to configure them at load balancer level. [https://thenewstack.io/deployment-strategies](https://thenewstack.io/deployment-strategies) has done good job at explaining what they are.

In this article I'm going to show how ingress-nginx can be used to do canary deployments. But the feature ingress-nginx provides can also be used to do Blue/Green deployment as well as A/B testing.

In the version of [`0.21.0`](https://github.com/kubernetes/ingress-nginx/releases/tag/nginx-0.21.0) ingress-nginx introduces a new [Canary feature](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary) that can be used to configure more than a single backend service for an ingress and few more annotations to also describe traffic distribution amongst the backend services. I'm going to show how to use that new feature with an example below. I'll first deploy a service and configure it's ingress. Then will deploy a new version of it under a different namesapce as canary. We will then use new ingress-nginx annotations to steer 10% of traffic to the canary backend. In the end of the article I'll also talk about how to always force requests to be proxied by canary service using a HTTP header or cookie and briefly explain how this feature can be used for configuring Blue/Green deployment and A/B testing.

We are going to use `echoserver` service as an example app from [https://github.com/kubernetes/ingress-nginx/blob/master/docs/examples/http-svc.yaml](https://github.com/kubernetes/ingress-nginx/blob/master/docs/examples/http-svc.yaml) and [minikube](https://kubernetes.io/docs/setup/minikube/) as local Kubernetes cluster.

Create production namespace:

```
> kubectl create ns echo-production
namespace "echo-production" created
```

Deploy the app:

```
> ~$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/docs/examples/http-svc.yaml -n echo-production
deployment.extensions "http-svc" created
service "http-svc" created
```

Here's the raw version of manifest applied above:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: http-svc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: http-svc
  template:
    metadata:
      labels:
        app: http-svc
    spec:
      containers:
      - name: http-svc
        image: gcr.io/kubernetes-e2e-test-images/echoserver:2.1
        ports:
        - containerPort: 8080
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP

---

apiVersion: v1
kind: Service
metadata:
  name: http-svc
  labels:
    app: http-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: http-svc
```

We will now expose the app using the below ingress manifest:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: http-svc
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: echo.com
    http:
      paths:
      - backend:
          serviceName: http-svc
          servicePort: 80
```

After saving the above YAML in `http-svc.ingress`, we do following to create the ingress:

```
> ~$ kubectl apply -f http-svc.ingress -n echo-production
ingress.extensions "http-svc" created
```

We now have the production version of our `echoserver` service running:

```
> ~$ curl -H "Host: echo.com" http://192.168.99.100:31988


Hostname: http-svc-686644794d-hv4rt

Pod Information:
	node name:	minikube
	pod name:	http-svc-686644794d-hv4rt
	pod namespace:	echo-production
	pod IP:	172.17.0.2

Server values:
	server_version=nginx: 1.12.2 - lua: 10010

Request Information:
	client_address=172.17.0.9
	method=GET
	real path=/
	query=
	request_version=1.1
	request_scheme=http
	request_uri=http://echo.com:8080/

Request Headers:
	accept=*/*
	host=echo.com
	user-agent=curl/7.54.1
	x-forwarded-for=172.17.0.1
	x-forwarded-host=echo.com
	x-forwarded-port=80
	x-forwarded-proto=http
	x-original-uri=/
	x-real-ip=172.17.0.1
	x-request-id=8055cb031471bfe034f4beaee7d7302b
	x-scheme=http

Request Body:
	-no body in request-
```

`http://192.168.99.100:31988` is obtained using:

```
> ~$ minikube service list
```

Now that we have production app running, I'm going to deploy canary version of the same service:

Create namesapce for the canary deployment:

```
> ~$ kubectl create ns echo-canary
namespace "echo-canary" created
```

Deploy the app under this namespace (in reality this should be a different - canary version of `echoserver` service, but for simplicity I'm deploying the exact app):

```
> ~$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/docs/examples/http-svc.yaml -n echo-canary
deployment.extensions "http-svc" created
service "http-svc" created
```

**Now comes the moment!** This is the step where we use Canary feature of ingress-nginx and configure load balancer to proxy 10% of traffic to the canary backend deployed above:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: http-svc
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
  - host: echo.com
    http:
      paths:
      - backend:
          serviceName: http-svc
          servicePort: 80
```

Create `http-svc.ingress.canary` with the above content and apply:

```
> ~$ kubectl apply -f http-svc.ingress.canary -n echo-canary
ingress.extensions "http-svc" created
```

`nginx.ingress.kubernetes.io/canary: "true"` tells ingress-nginx to treat this ingress differently and mark it as "canary". What this means internall is the controller is not going to try to configure a new Nginx virtual host for this ingress as it would normally do. Instead it will find the main ingress for this using host and path pair and associate this with the main ingress.

As you can guess `nginx.ingress.kubernetes.io/canary-weight: "10"` is what tells the ingress-nginx to configure Nginx in a way that it proxies 10% of total requests destined to `echo.com` to `echoserver` service under `echo-canary` namespace, a.k.a to the canary version of our app.


Now let's see whether it works as expected. To test this I'm goig to run a small script that sends 1000 requests and then will look at `pod namespace` in the response to count what percentage of total requests were processed by the canary version of our app.

Here's the script:

```
counts = Hash.new(0)

1000.times do
  output = `curl -s -H "Host: echo.com" http://192.168.99.100:31988 | grep 'pod namespace'`
  counts[output.strip.split.last] += 1
end

puts counts
```

And here are the outputs of two executions:

```
> ~$ ruby count.rb
{"echo-production"=>896, "echo-canary"=>104}
> ~$ ruby count.rb
{"echo-production"=>882, "echo-canary"=>118}
```

As you can see it worked as expected! Now if I edit the canary ingress and bump the weight up to `50`:

```
> ~$ ruby count.rb
{"echo-canary"=>485, "echo-production"=>515}
> ~$ ruby count.rb
{"echo-production"=>494, "echo-canary"=>506}
```

🎉

*Note* that `weight` is not a precise percentage, it's rather a probability at which a given request will be proxied to the canary backend service. Therefore in the above tests we see only approximate distributions.

### But how can I always manually force my request to hit canary backend service?

This can be useful for example when developers want to tophat their change. ingress-nginx provides two annotations for this: `nginx.ingress.kubernetes.io/canary-by-header` and `nginx.ingress.kubernetes.io/canary-by-cookie`. In this article I'm going to show how it's done using `nginx.ingress.kubernetes.io/canary-by-header`. To do so we need to edit the canary ingress and add following annotation:

```
nginx.ingress.kubernetes.io/canary-by-header: "you-can-use-anything-here"
```

Then when sending the request pass HTTP header `you-can-use-anything-here` set to `always`:

```
> ~$ curl -s -H "you-can-use-anything-here: always" -H "Host: echo.com" http://192.168.99.100:31988 | grep 'pod namespace'
	pod namespace:	echo-canary
```

If you want your request to be *never* proxied to the canary backend service then you can set the header to `never`:

```
> ~$ curl -s -H "you-can-use-anything-here: never" -H "Host: echo.com" http://192.168.99.100:31988 | grep 'pod namespace'
	pod namespace:	echo-production
```

When the value is absent or set anything other than `never`, or `always` then proxying by weight will be used. This works exactly the same for `nginx.ingress.kubernetes.io/canary-by-cookie`. Note that `canary-by-header` takes presedence over `canary-by-cookie`. In general the order of application of the rules is: `canary-by-header > canary-by-cookie > canary-weight`.


### Blue/Green deployment
This can be achieved by setting the weight to `0` or `100`. For example you can make your green deployment as main and
configure blue deployment's ingress to be canary. Initially you'll set the weight to `0` so no traffic will be proxied to blue deployment.
Once the new version of your app is ready, you deploy it as blue deployment and only after that set the weight to `100`, which will result in
steering all traffic away from green deployment to blue. And for the next deployment you can again deploy green and switch the weight back to `0` etc.

### A/B testing
This can be achieved using mix of business logic implemented in the backend service and `cookie-by-cookie` annotation.
For example let's say you want to show the new version of your app to only users under 30 years old. You can then configure

```
nginx.ingress.kubernetes.io/canary-by-cookie: "use_under_30_feature"
```

and implement a simple change in your backend service that for a given requests checks the age of signed-in user and if the age is under 30
sets cookie `use_under_30_feature` to `always`. This will then make sure all subsequent requests by those users will be proxied by the new
version of the app.
