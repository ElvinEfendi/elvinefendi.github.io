---
layout: post
title: "How can introducing a new request hop reduce the overall latency?"
date: 2023-06-17
comments: true
tags: reverse proxy, latency, performance, cdn
---

Intuitively, if you add a reverse proxy between A and B communicating with each other, the latency of communication should increase because of the overhead of the reverse proxy that terminates connections from A and opens new ones to B. In other words, every one connection becomes two connections. Moreoever, there's also processing overhead of the proxy that has to parse network payload based on the L7 protocol used, and re-encode it back when writing it to the upstream connection. So, one could easily fall into the trap of thinking that adding a new hop between A and B will always degrade the latency.

Below I describe a scenario where that's not the case, and in fact adding a new hop makes the overall latency better.

Imagine B nodes are your backend application servers (origin servers) deployed to handful of cloud regions, and A nodes are your edge servers deployed to many more regions to be closer to your users so that you can terminate connections closer to your users and offer them better service. If you would be using Cloudflare for your A nodes, the number of regions they are deployed would be over 200, and if you use GCP, it would be over 30 regions today. Either way, you would have several times more edge servers than the origin servers. You've already optimized the last mile latency by deploying your edge servers closer to your users, but what about the latency between your edge and origin servers? Below I'll reason that introducing an intermediary layer of reverse proxy between the edge and origin servers, would actually make response latency better!

Most of the edge servers will be geographically further away from the origin servers and your client requests will be dispersed across those edge servers, as a result each of the edge servers will get only small subset of your service's global traffic. This results in cold network path between edge and origin servers because there would not be enough traffic on the edge servers to keep connections to the origin servers alive. Add to it "treat your servers as cattle not pets" practice, which results in high churn rate on the origin side, connections would reset with each deploy. This means many new connection establishments between edge and origin servers. This degrades the latency because with many more edge servers than origin servers, it's highly likely that edge servers will be over 100ms RTT (round trip time) away from the origin servers. Assuming TLS v1.3 communication, this would mean at least 1.5 RTT for TCP handshake + 1 RTT for TLS handshake which is equal to 250ms. _As a result,
we will get worse latency because the new connection establishment is costly, and also because we would be doing many of them_.

We can solve this problem by strategically inserting a middle layer of reverse proxies between the edge and origin servers to fan in requests from edge servers and fan out them to the origin servers. The number of regions we would deploy the reverse proxies will be smaller than the number of edge servers, but higher than the number of origin servers. The reverse proxies would be closer to both edge and origin servers because they are in the middle. Assume they are 50ms (half of 100ms of total RTT if there were no reverse proxy) from edge and 50ms from origin servers. Even though we end up with 1 extra connections, math ends up being the same: `50ms x 2.5 + 50ms x 2.5 = 250ms`. So, by adding a new hop, we didn't make latency worse at least by introucing a new connection. But what about the overhead of reverse proxy itself? It has to decode/encode the payload right? Also, we were supposed to reduce the latency? That's where the magic of "economy of scale" comes in: since we have less number of reverse proxy regions, a given instance of reverse proxy would process more traffic than one edge server because it would fan in traffic from various edge origins. This results in less new connection establishments from reverse proxies to the origin servers. It also results in a given subset of edge servers talking to single reverse proxy (i.e using latency based traffic steering from edge to reverse proxies) vs all of the origin servers, which also increases the chances of a connection being kept alive. So also less number of new connections establishments from edge servers to the reverse proxies. Moreoever, since reverse proxies do not run our business logic, we also deploy them less, thus there's also less chance of connection resets. Depending on how the reverse proxies are placed, as a result, a lot of the times we would avoid the "at least 250ms" cost for routing single request from edge to origins servers, while adding at most 5ms reverse proxy processing time (this is usually a lot less overhead with high performance proxies such as Envoy or NGINX).

To conclude, by strategically placing a new layer of reverse proxies between edge and origin servers, in high traffic services, we can reduce average (depending on the traffic volume, chances are long tail latencies will decrease too) latency significantly at the cost of degrading the maximum latency by a few milliseconds.

There's also operational overhead of introducing a new reverse proxy service which one should also keep in mind when considering the above.