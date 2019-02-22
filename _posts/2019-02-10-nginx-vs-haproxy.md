---
title:  "Switching to HAProxy from Nginx"
date:   2019-02-21 00:00:00 -0600
categories: haproxy nginx comparison
---
# Overview
I'm currently in the process of switching my team's load balancers from Nginx to HAproxy. I mentioned it briefly in [in this blog post,](https://danielparker.me/haproxy/consul/srv/haproxy-srv-consul/) but I wanted to expand on some of my reasoning a bit more. Again, this isn't meant as a post bashing Nginx. I have had great success with Nginx and we still use it in certain areas. This is more of a post around the features HAProxy has that were compelling enough for me to switch.

For context, I want to talk a little bit about our architecture. We use Fastly as a CDN, and Fastly is the entry-point to our stack. From Fastly, we hit our edge load balancers (currently Nginx.) From the edge load balancers, we route to any number of backend microservices (depending on the environment, there could be hundreds.) Essentially, our edge load balancers make the decision on what backend to send the traffic to based on the route (like `/checkout/v2`) and handle the load balancing. They also encrypt everything with SSL and act as a line of defense against malicious calls.

# HAProxy features
There are 3 main features that HAProxy that Nginx doesn't (in the open source version, at least) that got me thinking about making a switch. I will go through each of these one at a time.

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

The specific line we care about is `  option httpchk GET /checkout/v2/health HTTP/1.1\r\nHost:\ haproxy`. This line tells HAProxy to call our backend with a request to `/checkout/v2/health` (with the request host as "haproxy".) This will proactively check for a 200 status code, and will mark the backend down immediately if the request fails. This is a great way to proactively remove an unhealthy backend. It also allows us to configure our health endpoint with additional checks - like waiting until the internal cache is warm or ensuring we can connect to our database before being marked as healthy. 

### TCP check
