---
title:  "Load Balancing Using Consul, SRV Records, and HAProxy"
date:   2018-9-21 00:00:00 -0600
categories: haproxy consul srv
---
# Overview
A few months ago, I wrote a [blog post](https://danielparker.me/nginx/consul-template/consul/nginx-consul-template/) about using Nginx + Consul in order to do dynamic routing and service discovery. Recently, I've started to explore the possibility of replacing Nginx with [HAProxy.](http://www.haproxy.org/) There are a few important reasons I wanted to use HAProxy rather than Nginx, and I want to cover those quickly before covering the main topic - using SRC records for routing and load balancing.

## Nginx vs HAProxy
Nginx has treated me very well. I've used Nginx in some form since 2012. This isn't meant to bash Nginx. Recently however, I've been needing/wanting some additional features. Nginx has most of these features, but unfortunately only in the Nginx Plus paid version. Let's take a look at the features:

* Upstream health check support
  * Nginx doesn't allow this in the free version
  * HAProxy can continuously health check its upstreams with a configurable URL to ensure they are still healthy
* Full metrics support
  * [Nginx Status Page](https://www.datadoghq.com/blog/how-to-collect-nginx-metrics/) is the only metrics available (without writing your own LUA or using someone else's)
  * HAProxy supports a [much fuller suite](https://www.datadoghq.com/blog/how-to-collect-haproxy-metrics/) of metrics and even a simple UI
* [SRV record](https://en.wikipedia.org/wiki/SRV_record) support
  * only available in Nginx Plus
  * available in HAProxy

These are the 3 main features that are currently driving my switch to HAProxy from Nginx. In this blog post - I'd like to focus on the last feature, SRV record support.

# SRV Records
SRV records are DNS records that contain more information about the host. Namely, the IP and the port are included. There also isn't really a limit on how many records can be returned through an SRV lookup - so if you have 50 backends, all 50 will be returned.

I've been looking in to using SRV records for a few years. One of the main drivers for this is the fact that [Consul](https://www.consul.io/docs/agent/dns.html#rfc-2782-lookup) supports SRV records. At my day job, we use Consul as our primary service discovery mechanism. This means every API, every database, everything is registered to Consul and can be discovered through DNS.

## Consul-Template
In our Nginx world, we use [Consul-Template](https://github.com/hashicorp/consul-template) to generate our Nginx configuration dynamically. This works well - and we use it in production. However, there are a few downsides:
* consul-template sometimes fails, and servers get out of sync
* consul-template must directly reload nginx for changes to take affect
* consul-template makes it harder to be flexible
  * because we're discovering 50+ services, loops are our friend - but this makes it hard to have unique config requirements between backends

So - for the reasons above I started researching SRV records in HAProxy. Let's take a look at some examples.

# HAProxy SRV Examples
Let's take a look at how to set up HAProxy to work with SRV records. The first requirement is to have a service registered to Consul. Let's take a quick look at setting up Consul:

## Consul configuration
I won't go in to installing Consul, it's [well-documented](https://www.consul.io/intro/getting-started/install.html) on the Hashicorp page. However, let's take a quick look at the [service](https://www.consul.io/docs/agent/services.html) JSON we want to discover:

```json
{
	"service": {
		"name": "ordering-service",
		"id": "ordering-service",
		"port": 9099,
		"tags": []
	}
}
```

This is a very simple example - but it allows us to do the following:
* register a service to consul with the name `ordering-service`
* register a port for that service, port `9099`

We're now ready to connect HAProxy to SRV records.

## HAProxy configuration
I won't go over installing HAProxy, it can be done through an RPM, or by following the [docs.](http://cbonte.github.io/haproxy-dconv/1.8/intro.html#3.6)

HAProxy is configured through an `haproxy.cfg` file. For the purpose of this blog, this file will contain our entire config.

I'm not going to post the full `haproxy.cfg` file for the sake of brevity, you can find examples [online](https://www.haproxy.org/download/1.8/doc/configuration.txt#2.5) and the RPM comes with an example.

The first part of the SRV configuration is the DNS resolver. We have to configure HAProxy to use Consul as its DNS server. The config looks like this:

```
resolvers consul
  nameserver consul $consul-client-ip:8600
  resolve_retries 30
  timeout retry 2s
  hold valid 120s
  accepted_payload_size 8192
```

This section sets up a DNS resolver named `consul` that we can use later on.

Next, we configure our backend. We'll use the `ordering-service` we set up earlier as our backend.

```
backend ordering-service
  mode http
  balance leastconn
  option httpchk GET /ordering-service/health HTTP/1.1\r\nHost:\ haproxy
  server-template ordering-service 50 _ordering-service._tcp.service.consul check resolvers consul
```

Let's drill in a bit on these configuration options.
* `mode http` just tells it to use http rather than tcp to connect
* `option httpchk...` sets up a health check. This tells HAProxy to continously hit a health endpoint to ensure it's healthy.
* `server-template...` sets up the SRV record.
  * ` _ordering-service._tcp.service.consul` is the SRV record. It must start with an underscore `_`, and the `_tcp` tells it to just pull all records. That can also be replaced with a tag from the Consul service.
  * `resolvers consul` tells it to use the consul resolver we set up previously.

This will allow HAProxy to automatically discover any server IP that is registered to Consul with the `ordering-service` ID. It also sends traffic to the port in the service, in our case `9099`. This is very useful since we can use dynamic ports in our services without having to list them in the HAProxy config file. This works really well using something like [Nomad]({% post_url 2017-09-01-nomad %}) too, since every service will bind to a dynamic port.

# Conclusion
That should be it - HAProxy should now find the service using the SRV revord. It will also continuously check the DNS record for any additions or subtractions to the service pool. You can add more servers with the `ordering-service` service configured, and HAProxy will automatically start to load balance across them.
