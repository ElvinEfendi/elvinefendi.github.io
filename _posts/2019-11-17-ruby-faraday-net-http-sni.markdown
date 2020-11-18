---
layout: post
title: "Custom SNI when making HTTP requests with Ruby net/http and Faraday"
date: 2020-11-17 17:30
comments: true
tags: ruby, net-http, faraday, tls, sni
---

The way Ruby's net/http library exposes customization of SNI is quite counterintuitive at the first sight.
It is also pretty much not documented. I discovered it through [this closed feature request](https://bugs.ruby-lang.org/issues/15215) 
and [the relevant source code](https://bugs.ruby-lang.org/issues/5180).

In this article I'm going to share two small code snippets on how to do that:
(1) with net/http directly (2) with Faraday gem. Let's assume our domain is `example.com`
and we want to force HTTPS requests to it to `forced-endpoint.com` server.

#### Directly with net/http

```
require 'net/http'

uri = URI('https://example.com')

Net::HTTP.start(uri.host, uri.port, use_ssl: true, ipaddr: 'forced-endpoint.com') do |http|
  request = Net::HTTP::Get.new(uri)
  response = http.request(request)

  p response.code
  p response.each_header.to_h.to_s
end
```

#### Using Faraday gem

```
require 'faraday'


conn = Faraday.new('https://example.com') do |f|
  f.adapter :net_http do |http|
    http.ipaddr = 'forced-endpoint.com'
  end
end

response = conn.get('/')
p response.status
```

Both code snippet will result in TCP connection to `forced-endpoint.com`, where TLS handshake is done
using `example.com` as SNI while HTTP Host header is set to `example.com`.

In both cases the key is setting `ipaddr` attribute. One can also set it to a specific IP address.
That is where the counterintuitiveness of this (at the first sight) configuration comes from:
you have to mention address for TCP/socket layer in order to force net/http to use provided host as SNI.
What would _seemingly_ be more intuitive is to explicitly set SNI, for example that is how Go does it
using `TLSClientConfig.ServerName` attribute. However if you think about it a bit more, Ruby's approach is arguably better!
The author of the related patch explains it in [this comment](https://bugs.ruby-lang.org/issues/15215#note-5) as to why
he chose to expose overriding of `ipaddr` for underlying TCP connection instead of the SNI.
The tldr; is that when you override SNI, most of the times you also want to override HTTP `Host` header
so that they are the same - that is also how `curl` with its `--resolve` argument works.
