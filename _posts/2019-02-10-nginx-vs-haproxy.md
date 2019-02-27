---
title:  "Switching to HAProxy from Nginx"
date:   2019-02-21 00:00:00 -0600
categories: haproxy nginx comparison
---
# Overview
I'm currently in the process of switching my team's load balancers from Nginx to HAproxy. I mentioned it briefly in [in this blog post,](https://danielparker.me/haproxy/consul/srv/haproxy-srv-consul/) but I wanted to expand on some of my reasoning a bit more. Again, this isn't meant as a post bashing Nginx. I have had great success with Nginx and we still use it in certain areas. This is more of a post around the features HAProxy has that were compelling enough for me to switch.

For context, I want to talk a little bit about our architecture. We use Fastly as a CDN, and Fastly is the entry-point to our stack. From Fastly, we hit our edge load balancers (currently Nginx.) From the edge load balancers, we route to any number of backend microservices (depending on the environment, there could be hundreds.) Essentially, our edge load balancers make the decision on what backend to send the traffic to based on the route (like `/checkout/v2`) and handle the load balancing. They also encrypt everything with SSL and act as a line of defense against malicious calls.

# HAProxy Advantages
There are 3 main features that HAProxy has that Nginx doesn't (in the community version, at least) that got me thinking about making a switch. I will go through each of these one at a time.

## Upstream health check support
The first feature that is important to me is upstream health checks. Nginx has a health check feature in Nginx Plus, but that isn't an option for me. Nginx can also retry when the upstream call fails, but there isn't really a proactive way for Nginx to check the backend health without using Nginx Plus. HAProxy *does* have health check support, in fact, it supports TCP and HTTP health checks.

The docs offer multiple options for configuring, but we usually set our health checks directly in our HAProxy backend.

### HTTP check
In our backend, you can define an HTTP check like this:

```
backend checkout-v2
  mode http
  balance roundrobin
  option httpchk GET /carts/v4/actuator/health HTTP/1.1\r\nHost:\ haproxy
  server-template checkout-v2 10 checkout-v2.service.consul:8080 resolvers myresolver resolve-prefer ipv4
```

The specific line we care about is `option httpchk GET /checkout/v2/health HTTP/1.1\r\nHost:\ haproxy`. This line tells HAProxy to call our backend with a request to `/checkout/v2/health` (with the request host as "haproxy".) This will proactively check for a 200 status code, and will mark the backend down immediately if the request fails. This is a great way to proactively remove an unhealthy backend. It also allows us to configure our health endpoint with additional checks - like waiting until the internal cache is warm or ensuring we can connect to our database before being marked as healthy.

### TCP check
HAProxy also allows standard TCP checks. These checks are less flexible than HTTP - it can only tell you if the IP:Port combination is open and listening - but it's a good start if you don't have/aren't ready for HTTP health checks.

```
backend checkout-v2
  mode http
  balance roundrobin
  server-template checkout-v2 10 checkout-v2.service.consul:8080 check port 8080 resolvers myresolver resolve-prefer ipv4
```

The important part of this stanza is `check port 8080`. This tells HAProxy to check port 8080 on each backend IP periodically and ensure it's listening before routing traffic.

---

There are many more options in the HAProxy documentation. Check out the [TCP check options](https://cbonte.github.io/haproxy-dconv/1.9/configuration.html#4.2-tcp-check%20connect) or the [HTTP check options.](https://cbonte.github.io/haproxy-dconv/1.9/configuration.html#option%20httpchk)

## Full Metrics Support
Metrics have become very important for everyone to diagnose problems and understand what is happening with your infrastructure. Nginx, unfortunately, does not have the same level of metrics support as HAProxy. Nginx (outside of Nginx Plus) only offers [basic stats](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html) out of the box. In the past, we actually wrote some scripts to parse the Nginx logs and convert them to metrics (like # of 200 status codes, latency, etc.) - but this become unmanageable as our traffic levels grew. HAProxy has full metrics support in the free version. Here is a quick overview of the [available metrics.](https://cbonte.github.io/haproxy-dconv/1.9/management.html#9.3-show%20stat)

We were able to point Telegraf at the HAProxy metrics socket and easily get useful information about our frontends and backends. Datadog has a great post on getting everything set up with HAProxy and starting to collect metrics, I recommend reading [this.](https://www.datadoghq.com/blog/how-to-collect-haproxy-metrics/)

## DNS SRV Record Support
At Target, we heavily use [Consul](https://www.consul.io/) as our service registry. One of the benefits of Consul is that every service registered to it can be addressed using DNS. This includes [SRV records](https://en.wikipedia.org/wiki/SRV_record), which also have the port. We've started to use [Nomad](https://www.hashicorp.com/resources/nomad-scaling-target-microservices-across-cloud) as a scheduler - so the ports of our microservices are dynamic. In order to keep our routing as simple and as up-to-date as possible, I wanted to use these SRV records in our load balancers.

Nginx does not support SRV in the community version. It only supports SRV records in Nginx Plus. HAProxy, however, does support SRV records. I wrote a post a few months ago on [configuring HAProxy to work with SRV records.](https://danielparker.me/haproxy/consul/srv/haproxy-srv-consul/) This was hugely beneficial to us as we could very easily integrate Nomad and it's dynamic ports with our frontend load balancers.

# Conclusion
Nginx has treated me very well in the past, but I'm currently switching most of my workloads over to HAProxy. There are still places where I use Nginx, and probably always will. The features listed above were too good to pass up, so I've started using HAProxy in the majority of my deployments. I look forward to continuing to learn all the capabilities of HAProxy.
