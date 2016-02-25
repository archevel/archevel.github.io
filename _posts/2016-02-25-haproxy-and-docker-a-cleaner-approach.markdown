---
layout: post
title:  "Haproxy and docker with auto request routing - a cleaner approach"
date:   2016-02-25 08:25:00 +0000
categories: docker docker-compose haproxy confd
---

[Last post about haproxy and docker][ha-docker-hack] conianed a hack that allowed us to run a container with haproxy that automatically routed to linked containers. My main gripe with that approach was that it used the old default docker bridge network mode. Newer vesions of docker have allows us to better isolate our containers by creating [user defined networks][docker-networks]. This new feature also comes with a built in dns that the conatiner can use to look up other services connected to the same network. A simple example with docker compose would be:

```YAML
version: '2'

services:
  # A simple flask application
  genie:
    image: genie:XYZ
    networks:
      - app_net
    
  proxy:
    image: haproxy:1.6
    networks:
      - app_net
    ports: 
      - "80:80"
    environment:
      appservers: genie

networks:
  app_net:
    driver: bridge
```

The top level `networks` entry will generate a bridged network called `<folder>_apps_net` to which both the `genie` and `proxy` services will be connected. However, now proxy will no longer have a `/etc/hosts` entry for genie. Therefore the environment variable `appservers` has been added to proxy. Above it has a single entry `genie`, but if there were more than one we can simply use a colon serparated list of hostnames. We still need to generate our haproxy configuration. We could put our bash hats on again, but I realy didn't like using `sed` to generate output since it was a quite opaque. Instead the [templating tool confd][confd] is an excellent choice in this case. It is a tiny go binary that can use values from environment variables, consul or etcd to fill out templates.

Confd needs a toml file indicating where the template is, what variables it should use for the template and where to output the result.

```Shell
[template]
src = "haproxy.conf.tmpl"
dest = "/etc/haproxy.conf"
keys = [
  "/appservers"
]
```

The above is the contents of `/etc/confd/conf.d/haproxy.toml` it is accompanied by the `haproxy.conf.tmpl` file specified by `src` which by default should be located in `/etc/confd/templates/`. One more thing to note about the keys (at least for environment variables) is if your environment variable contains `_` confd will replace those with `/`, e.g. `SOME_VARIABLE_FOO` would have the confd key `"/some/variable/foo"`. The template file looks similar to what we had in the last post, but with some additions:

```Shell {% raw %}
global
    maxconn 256

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

resolvers dns
  resolve_retries       5
  timeout retry         10s
  hold valid 10s

frontend http-in
    bind *:80
    redirect code 301 prefix / drop-query append-slash if { path_reg ^/[^/]*$ }
{{ $nodes := split (getenv "appservers") ":" }}{{range $nodes}}
    use_backend {{.}}_backend if { path_beg /{{.}}/ }
{{end}}
{{ $nodes := split (getenv "appservers") ":" }}{{range $nodes}}
backend {{.}}_backend
     reqrep ^([^\ ]*\ /){{.}}[/]?(.*)     \1\2
     server {{.}}_server {{.}}:5000 resolvers dns check inter 1000 
{{end}}
{% endraw %}```

There is a new `resolvers` entry and towards the end after the `frontend` entry we see confd's templating. The `resolvers` entry is necesseary to define how haproxy should do dns lookups for ur container names in the generated `backend` entries.

Looking closer at the below templating will help illustrate how we generate teh final config:

```Shell {% raw %}
{{ $nodes := split (getenv "appservers") ":" }}\{{range $nodes}}
    use_backend \{{.}}_backend if { path_beg /\{{.}}/ }
{{end}}
{% endraw %}```

Here we ask confd with `(getenv "appservers")` to fetch the value of the config key `"appservers"`. We split this value on `":"` (i.e. when we have more than one service to route requests to) and assign it that to the `$nodes` variable. We then iterate over `$nodes` with `{% raw %}{{range $nodes}}{% endraw %}`. Finally we use `{% raw %}{{.}}{% endraw %}` to print out the current value. If the `appserver` environment variable had the value `genie:foo:bar` the above would generate:

```Shell
    use_backend genie_backend if { path_beg /genie/ }
    use_backend foo_backend if { path_beg /foo/ }
    use_backend bar_backend if { path_beg /bar/ }
```

Each mapping entry here get a corresponding backend entry generated in the same way through the template.

All that remains is to change the haproxy containers start command to actually run confd. Using the below `configure-and-start.sh` script causes confd to run and then haproxy to start with the generated configuration:

```Shell
#!/bin/bash
set -e

/opt/confd -onetime -backend env

echo Starting haproxy!
haproxy -f /etc/haproxy.conf
```

All in all this feels like a nicer aproach since it uses plain dns to resolve the services. This allows services to be restarted in contrast with the previous solution where each service's IP was "hard coded" when starting the haproxy container. In addition if we end up storing the appservers list in consul or etcd there is very little change needed. Finaly if we switch to using and `overlay` network spanning multiple docker hosts this solution will still work without modification!