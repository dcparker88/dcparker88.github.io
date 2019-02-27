---
title:  "Simple Blue/Green Deployments with Nomad and HAProxy"
date:   2019-02-21 00:00:00 -0600
categories: haproxy blue-green deployments canary nomad
---
# Overview
I've recently started deploying HAProxy to replace Nginx for most of our app load balancing. You can read more about my decision to switch from Nginx to HAProxy [in this blog post.]() One reason I am switching is because of DNS SRV record support, brought on by our use of [Nomad at Target.](https://www.hashicorp.com/resources/nomad-scaling-target-microservices-across-cloud) Another feature Nomad gives us is blue/green and canary deployments. I needed to figure out how to integrate these features with our edge load balancer - HAProxy.

## Nomad
Nomad gives us the ability to do blue/green and canary [deployments](https://www.nomadproject.io/guides/operating-a-job/update-strategies/blue-green-and-canary-deployments.html) Nomad differentiates "live" traffic from "canary" (or blue/green) by using Consul tags. For example, we may have 4 microservices deployed that are active. These would have an `live` [tag](https://www.consul.io/docs/agent/services.html) in Consul. If we deployed a canary, a 5th microservice would be deployed with a `canary` tag. You can see this configuration in our Nomad job file:

```
job "canary-deployments" {
  type = "service"
  update {
    stagger      = "30s"
    max_parallel = 1
    min_healthy_time = "120s"
    canary = 1
  }
...
service {
  port = "http"
  name = "canary-deployments"

  tags = [
    "live"
  ]

  canary_tags = [
    "canary"
  ]
}
```

The fist piece `canary = 1` tells Nomad to enable canary deployments. The second, `tags = [ "live" ]`, tags everything currently running with the `live` tag. The last, `canary_tags = [ "canary" ]`, tags any ongoing canary deployment with the `canary` tag. These tags are important, as they now allow us to route specific requests to the proper backend using HAProxy.

## HAProxy
Now we want to set up HAproxy to properly route us to the backend we expect. In the simplest configuration, we'll have 2 backends: the live backend, and the canary backend. Let's take a peek at how that looks:

```
frontend http-in
  mode http
  bind *:80

backend api-v1
  mode http
  server-template api-v1 10 live.api-v1.service.consul check resolvers myresolver resolve-prefer ipv4

backend api-v1-canary
  mode http
  reqrep ^([^\ :]*)\ /canary/(.*)     \1\ /\2
  server-template api-v1-canary 10 canary.api-v1.service.consul resolvers myresolver resolve-prefer ipv4

```

This sets up 3 things. A frontend in HAProxy that is listening for traffic on port 80. It also sets up 2 backends we can route traffic to - `api-v1` and `api-v1-canary`. Since we're using Consul DNS records to generate the backend list, we can also use Consul tags. Backend `api-v1` will only find services registered with the `live` tag, and `api-v1-canary` will only find backends with the `canary` tag. We also add an option, `reqrep ^([^\ :]*)\ /canary/(.*)     \1\ /\2` - this will strip off the `/canary/` (we'll cover this later) so we don't pass it to our backend. That way we don't have to tell our APIs to look for a `/canary/` path - to the API all requests are the same.

### Routing Based On Request Path
So now, let's talk about routing to these backends. The first option is to route based on the request path. We can have the following config:

```
acl url_api-v1 path_beg /api/v1/
use_backend api-v1 if url_api-v1

acl url_api-v1-canary path_beg /canary/api/v1/
use_backend api-v1-canary if url_api-v1-canary  
```

This is fairly simple - it checks the URL for a path match, and sends it to the proper backend. So, if we made the call `http://haproxy/api/vi` it would route to the active backend, `api-v1`. Alternatively, if we made the call `http://haproxy/canary/api/v1` it would route to the canary backend `api-v1-canary`, allowing us to test the new version we're deploying before it goes live.

### Routing Based On A Header
We can also make routing decisions based on a request header. This is nice because we don't have to change the paths of the API or worry about stripping off extra paths, but can require a larger code change than just changing a URL. Let's look at a simple example:

```
acl hdr_api-v1 hdr_val(My-Custom-Header) eq api-v1
use_backend api-v1 if hdr_api-v1

acl hdr_api-v1-canary hdr_val(My-Custom-Header) eq api-v1-canary
use_backend api-v1-canary if hdr_api-v1-canary
```

This tells HAProxy to check for the value of the header `My-Custom-Header`. If the value is `api-v1` it will send you to that backend. If the value is `api-v1-canary` it will send you to the canary backend. This is a very simple example - check the HAProxy [ACL documentation for more.](https://www.haproxy.com/blog/introduction-to-haproxy-acls/)

# Conclusion
This is a simple way to set up blue/green and canary deployments using Nomad and HAProxy. I am continuing to learn and find new ways to manage traffic with HAProxy, so I will continue to post options!
