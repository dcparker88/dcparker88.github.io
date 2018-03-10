---
title:  "Generating dynamic config with Nginx and Consul-Template"
date:   2018-3-05 00:00:00 -0600
categories: nginx consul-template consul
---
# Overview
In my day job, we heavily use Nginx for our edge web servers. These servers route traffic to many different microservices. This means the Nginx config can become complex - with many different Nginx [locations](http://nginx.org/en/docs/http/ngx_http_core_module.html#location) and [upstreams.](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream) Recently, we started registering all of these services in [Consul.](https://www.consul.io) This gives us many benefits - one of which is the ability to use [consul-template.](https://github.com/hashicorp/consul-template) This tool allows us to dynamically generate code based on the services registered inside of Consul. This means we can automatically add servers to Nginx as new ones come online - and automatically remove them as we scale down or a service fails. I wanted to give some examples around how we accomplished this, and what our configuration looks like.

## Background
First, let's start with the Nginx configuration. I won't get in to the full Nginx config - if anyone is interested I can put that in another post. For now, let's just focus on the location and upstream block of Nginx.

Each of our microservices has a unique route - usually something like `/$api-name/$version`. This means the Nginx location block needs to know the unique route for each microservice it's responsible for routing. Each location block has a corresponding upstream block. This upstream block needs to know every server that exists that is servicing traffic for the API matching the location. So for high-traffic APIs, we may have 50 or more servers in the upstream block, each serving traffic. Let's take a look at what each of these blocks looks like:

`nginx-locations.conf`

{% highlight nginx %}
location /dwave-scheduler {
    proxy_pass http://dwave-scheduler-pool/dwave-scheduler;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
}

location /order-api {
    proxy_pass http://order-api-pool/order-api;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
}

location /order-transfer {
    proxy_pass http://order-transfer-pool/order-transfer;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
}

location /session/session {
    proxy_pass http://session-session-pool/session/session;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
}
{% endhighlight %}


As you can see above, there are 4 distinct APIs. Each has a "pool" name (the upstream block below) and a unique route. Let's take a look at the upstreams:

`nginx-upstreams.conf`

{% highlight nginx %}
upstream dwave-scheduler-pool {
  least_conn;
  keepalive 32;
  server 172.22.0.1:8080;
  server 172.22.0.2:8080;
  server 172.22.0.3:8080;
}

upstream order-api-pool {
  least_conn;
  keepalive 32;
  server 172.22.0.4:8080;
  server 172.22.0.5:8080;
  server 172.22.0.6:8080;
  server 172.22.0.7:8080;
  server 172.22.0.8:8080;
  server 172.22.0.9:8080;
}

upstream order-transfer-pool {
  least_conn;
  keepalive 32;
  server 172.22.0.10:8080;
  server 172.22.0.11:8080;
  server 172.22.0.12:8080;
  server 172.22.0.13:8080;
  server 172.22.0.14:8080;
  server 172.22.0.15:8080;
  server 172.22.0.16:8080;
}

upstream session-session-pool {
  least_conn;
  keepalive 32;
  server 172.22.0.17:8080;
  server 172.22.0.18:8080;
{% endhighlight %}

Each unique API has a set of servers that are serving traffic for that API. This can change any time - maybe we need to scale up to handle more traffic, or maybe one of these servers crashes.

In the past - this would have been configured statically. We would have to manually go and update it (or let something like Chef do it) each time a server changed. It also wasn't all that resilient - if one of the servers above crashed, it could take 30 minutes or more for Chef to update the config, and that's if Chef actually knew the API wasn't healthy. Enter consul-template.

## Consul-Template
As our service catalog grew, and we started deploying more and more frequently, we realized we needed this Nginx configuration to become more dynamically. Since we were already moving to register all of our services in Consul, Consul-Template seemed like a natural fit. I won't go too far in to what consul-template is - the Hashicorp website does a great job of that. Basically - consul-template allows us to dynamically generate configuration files from services registered to Consul. We now have a consul template file `.ctmpl` for each of our config files - in this case, one for the `locations` block and one for the `upstreams` block. Let's take a look at each one:

`nginx-locations.ctmpl`

{% highlight text %}
{% raw %}
  {{- range services -}}
    {{- if in .Tags "nginx-route" -}}
      {{- $boxes := service .Name }}
        {{- if gt (len $boxes) 0 -}}
  location /{{.Name | replaceAll "--" "/"}} {
      proxy_pass https://{{.Name | replaceAll "--" "-"}}-pool/{{.Name | replaceAll "--" "/"}};
      proxy_http_version 1.1;
      proxy_set_header Connection "";
  }
        {{- end -}}
    {{- end -}}
  {{- end -}}
  {% endraw %}
{% endhighlight %}

Let's step through that. The first block tells consul-template to get a list of all the services registered in Consul. We then filter by a specific tag, `nginx-route`, so we know it's a service we specifically want to add to nginx. This means any API registering with Consul that wants to be routed through Nginx needs this tag. The next 2 lines are a quick check to make sure the service actually has healthy boxes - if it doesn't, Nginx will complain that there is an upstream block with no hosts in it.

The next part is a little confusing. We set up the unique route by using the service name, and replacing any `--` with a `/`. This allows us to dynamically generate the route based on the service. If an API, say `/order-api/v2` wanted to be in Nginx, it would register itself with the name `order-api--v2` and the tag `nginx-route`. Nginx would then know to route this API, and route it to the route `/order-api/v2`.

The next part is how we link the upstream block to the location block. We keep the same naming convention - except we replace the `--` with a `-` for the pool name to make sure we don't add an extra route. This matches the name in the upstream block - so anything coming to `/order-api/v2` will get routed to the `order-api-v2-pool`. Let's take a look at the upstream block:

`nginx-upstreams.ctmpl`

{% highlight text %}
{% raw %}
  {{- range services -}}
    {{- if in .Tags "nginx-route" -}}
      {{- $boxes := service .Name }}
        {{- if gt (len $boxes) 0 -}}
  upstream {{.Name | replaceAll "--" "-"}}-pool {
    least_conn;
    keepalive 32;
    {{- range service .Name }}
    server {{.Address}}:{{.Port}};{{ end }}
  }
        {{- end -}}
    {{- end -}}
  {{- end -}}
  {% endraw %}
{% endhighlight %}

Very similar to the location block - except in this one we step through the specific service, and list out the address and port of each one. This means that any time a new box is added, or a [consul health check](https://www.consul.io/intro/getting-started/checks.html) fails, it will automatically update Nginx with the new configuration file.

# Conclusion
We've been using this in production for the last few months, and it works very well. We now have the following benefits:
* automatic discovery of new microservices
* automatic discovery of servers
  * failed servers are removed
  * added servers are added

I'm continuing to learn Nginx and Consul, so if anyone has any suggestions, let me know!
