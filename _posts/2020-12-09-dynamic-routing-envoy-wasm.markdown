---
layout: post
title: "Extending Envoy with WASM to do dynamic (multi-region edge) routing"
date: 2020-12-09 20:30
comments: true
tags: envoy, wasm, plugin, routing
---

Recently I got a chance to try out a newly introduced Envoy plugin system.
One can now extend Envoy using Webassembly (WASM). It took me some time to put the
pieces together as the plugin system is under active development and public
examples and documentation can become obselete any time (yes, this one too).
I found the [integration test data](https://github.com/envoyproxy/envoy/tree/master/test/extensions/filters/http/wasm/test_data)
to be the most helpful and up-to-date. You can read more about the new system [here](https://github.com/proxy-wasm/spec/blob/master/docs/WebAssembly-in-Envoy.md).

In order to experiment with this new plugin system I built an Envoy WASM plugin
that for a given request:

1. It extracts Host header, and makes an async HTTP request to an external discovery service passing the Host header as a query string
2. When the async request finishes, it reads the JSON response, extracts the value of `region` field from the JSON response
3. Then it sets that value to `region` HTTP header in the original request.

I also configure Envoy's routing using `cluster_header` router which makes sure Envoy dynamically picks the cluster based on
the value of `region` HTTP request header, here's the corresponding configuration:

```
- name: envoy.filters.http.router
route_config:
  name: local_route
  virtual_hosts:
  - name: local_service
    domains: ["*"]
    routes:
    - match: { prefix: "/" }
      route:
        cluster_header: region
```

This ensures that NGINX routes a given request to the cluster set as the value of `region` HTTP header.
You have to make sure that all potential values of that header are configured as clusters.

I also have following clusters configured:

```
clusters:
- name: us_east1
  connect_timeout: 5s
  type: LOGICAL_DNS
  dns_lookup_family: V4_ONLY
  load_assignment:
    cluster_name: us_east1
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: httpbin.org
              port_value: 443
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
      sni: httpbin.org
- name: us_central1
  connect_timeout: 5s
  type: LOGICAL_DNS
  dns_lookup_family: V4_ONLY
  load_assignment:
    cluster_name: us_central1
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: www.elvinefendi.com
              port_value: 443
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
      sni: www.elvinefendi.com
- name: discovery_service
  connect_timeout: 30s
  type: LOGICAL_DNS
  dns_lookup_family: V4_ONLY
  load_assignment:
    cluster_name: discovery_service
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: httpbin.org
              port_value: 443
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
      sni: httpbin.org
```

So, given a request, depending on HTTP Host header it will either be routed to `us_east1` or `us_central` cluster.

Now that we have necessary Envoy configuration, we can start building the plugin.

Here is the complete source code:

```
#[derive(Deserialize)] struct HttpBinResponse { args: HttpBinResponseArgs }
#[derive(Deserialize)] struct HttpBinResponseArgs { region: String }

#[no_mangle]
pub fn _start() {
    proxy_wasm::set_log_level(LogLevel::Trace);
    proxy_wasm::set_http_context(|_, _| -> Box<dyn HttpContext> {
        Box::new(RegionalRouter)
    });
}

struct RegionalRouter;

impl Context for RegionalRouter {
    fn on_http_call_response(&mut self, _: u32, _: usize, body_size: usize, _: usize) {
        if let Some(body) = self.get_http_call_response_body(0, body_size) {
            let data: HttpBinResponse = serde_json::from_slice(body.as_slice()).unwrap();

            info!("Routing to the region: {}", data.args.region);

            // NOTE: in the envoy.yaml we configured the route with `cluster_header: region`,
            // which means it will pick the cluster from the value of `region` HTTP header.
            // So here we get the region from our service discovery for given host, and set the
            // region header based on that. The rest should be taken care by Envoy.
            self.set_http_request_header(&"region", Some(&data.args.region));

            // NOTE: for now routing table is not flushed automatically,
            // this is a workaround. See https://github.com/proxy-wasm/spec/issues/16.
            self.clear_http_route_cache();

            self.resume_http_request();

            return
        }
    }
}

impl HttpContext for RegionalRouter {
    fn on_http_request_headers(&mut self, _: usize) -> Action {
        // NOTE: normally this or similar logic would be part of
        // the discovery service, but we are merely mocking using httpbin.org.
        let host = self.get_http_request_header(":authority").unwrap();
        let hash_code = (host.chars().next().unwrap() as u32) % 2;
        let mut expected_region = "us_east1";
        info!("Hash code of {} is {}", host, hash_code);
        if hash_code == 1 {
            expected_region = "us_central1";
        }

        self.dispatch_http_call(
            "discovery_service",
            vec![
                (":method", "GET"),
                (":path", &format!("/anything?region={}", expected_region)),
                (":authority", "httpbin.org"),
            ],
            None,
            vec![],
            Duration::from_secs(5),
        ).unwrap();

        Action::Pause
    }

    fn on_http_response_headers(&mut self, _: usize) -> Action {
        self.set_http_response_header("Processed-By", Some(&"Regional Router"));

        Action::Continue
    }
}
```

The main logic of our plugin happens in `on_http_request_headers` and in `on_http_call_response` callbacks.
In `on_http_request_headers` we issue an async request to a discovery service (ignoring my mocks to be able to use httpbin.org as a discovery service)
and pause the main request. An ability to issue an async request already distinguishes Envoy's WASM filter from Lua filter because
doing an HTTP call with Lua in Envoy is blocking. However in this case Envoy thread can still process other requests while waiting for this non-blocking async request.

Then in `on_http_call_response` callback that gets called when the async request completes, we read response body and extract `region` that the given request needs be
routed to. After that we set its value to `region` HTTP header in the original request. The rest is taken care by Envoy.
According to [this response](https://github.com/proxy-wasm/spec/issues/16#issuecomment-740506918) Envoy WASM will eventually provide an API to
dynamically modify routing decision instead of indirectly doing it like this. Finally we resume the original request.

I have the whole setup published in [https://github.com/ElvinEfendi/envoy-wasm-regional-routing](https://github.com/ElvinEfendi/envoy-wasm-regional-routing).
It includes a fully working setup using Docker compose.
