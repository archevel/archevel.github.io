---
layout: post
title:  "Automatically route requests to containers linked with Haproxy"
date:   2016-02-16 07:59:54 +0000
categories: docker docker-compose haproxy 
---

If you have multiple containers running on a server that expose webbapps you might want to consider putting a proxy in front of them. One way to do that is to use [Haproxy][haproxy]. Getting started with it if you are already using docker is simple since there's [an official docker image for it][haproxy-image]. For small deployments docker-compose is a nice way to organise the containers. As an example we'll use this compose file:

```
version: '2'

services:
  # A simple flask application
  genie:
    image: genie:XYZ
    network_mode: "bridge"
    
  proxy:
    image: haproxy:1.6
    network_mode: "bridge"
    ports: 
      - "80:80"
    links:
      - genie

```

Genie is a pyhon flask applicatin and proxy is our haproxy container. Ideally we'd want haproxy to automatically detect the genie container and route HTTP requests to it. We will start with this haproxy config file (found at `/etc/haproxy.conf.tmpl` in the container): 

```
global
    maxconn 256

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend http-in
    bind *:80
    redirect code 301 prefix / drop-query append-slash if { path_reg ^/[^/]*$ }

```

Here we basically have a frontend that bound to port 80 and that currently will redirect "malformed" paths by appending a slash to them, e.g. `http://localhost/no-final-slash` will be redirected to `http://localhost/no-final-slash/` that way the webapps can more easily use relative paths to resources and is not strictly necessary. The documentation for [haproxy configuration can be found here][haproxy-config] (there is lots of things one can do).

This config lacks any backends so we need to add them when we bring up the haproxy container. To achieve this we'll use the fact that docker's bridged network adds entries to the linking container's `/etc/hosts` file. So let's put our bash hats on!

```
#!/bin/bash
set -e

CONF_FILE=$(</etc/haproxy.conf.tmpl)
echo "CONF_FILE has original content:"
echo "#####"
echo "$CONF_FILE" | tee /usr/local/etc/haproxy/haproxy.conf
echo "#####"
```
This reads the .tmpl file and echos the content. `tee` is a nice program that takes stdin and push it both to stdout and to an outfile. Next we grep the relevant host entries from the `/etc/hosts` file 

```
echo 
echo "Grabing docker containers from the hostfile (using docker's subnet)"
HOSTS=$(grep -P "^172.17.[.0-9]+\t[^ _]+ .+$" /etc/hosts)
echo "HOSTS to specify backends and mappings for:"
echo "#####"
echo "$HOSTS" 
echo "#####"
```
The grep command here finds the rows that are on the docker subnet (172.17.x.y) and that do not have a `_` in their name. This is because docker-compose adds some extra entries. When started in the folder `foo` the entries `foo_genie_1`, `genie_1` and `genie` would all added and I only want my mappings to only contain the container name `genie`. Note that this won't work if the container has a underscore as part of the name! But, no one likes the underscore character anyway :).

Btw, all the echoing is only a nice to have. Since this is run on startup of the haproxy container it will show up in the docker logs which makes it easier to debug. Next up: sed-fu! 

```
echo
echo "Using sed to create mappings entries for the found hosts"
MAPPINGS=$(sed "s/\(^[.0-9]*\)\t\([^ _]*\) .*$/    use_backend \2 if \{ path_beg \/\2\/ \}\n/g" <<< "$HOSTS")
echo "Appending these MAPPINGS to conf file:"
echo "#####"
printf "$MAPPINGS\n\n" | tee -a /usr/local/etc/haproxy/haproxy.conf
echo "#####"
```
Here sed is used with the input from the hosts we grepped out earlier we use it to produce mappings which are then appended to  the haproxy.conf file again using `tee`. This transforms a host entry like:

```
172.17.0.3	genie 0c984672792c labs_genie_1
```

to:

```
    use_backend genie if { path_beg /genie/ }
```

So if a path start with the name of the container we use the backend for that container, which we will generate next!

```
echo 
echo "Using sed to create the backend entries for the found hosts"
BACKENDS=$(sed "s/\(^[.0-9]*\)\t\([^ _]*\) .*$/backend \2 \n    reqrep ^([^\\\ ]*\\\ \/)\2[\/]?(.*)     \\\1\\\2\n    server \2 \1:5000 maxconn 32\n/g" <<< "$HOSTS")
echo "Appending these BACKENDS to conf file:"
echo "#####"
echo "$BACKENDS" | tee -a /usr/local/etc/haproxy/haproxy.conf
echo "#####"
```

The sed commands search part above is identical to the previous one (we still want the simplest container names and their IP). The replace part is a bit more involved and easy to get wrong. There's a bunch of escaping mixed in with the replacements and we also create a new regex to boot! At this point you might appreciate why I've been using `tee` ;) In any case the sed command would transforms the host entry:

```
172.17.0.3	genie 0c984672792c labs_genie_1
```

to:

```
backend genie 
     reqrep ^([^\ ]*\ /)genie[/]?(.*)     \1\2
     server genie 172.17.0.3:5000 maxconn 32
```

Now we just instruct the container that it should start haproxy with the generated config file and all the containers (in this case only `genie`) linked to proxy will be accessible on a subpath, e.g. `http://localhost/genie`. The final script can be found in [this gist][configure-and-start.sh], which include the actual startup of haproxy. It can be used as the start command of a haproxy image based on the [official image][haproxy-image] (version 1.6).

[haproxy]: http://www.haproxy.org/
[haproxy-image]: https://hub.docker.com/_/haproxy/
[haproxy-config]: http://cbonte.github.io/haproxy-dconv/configuration-1.6.html
[configure-and-start.sh]: https://gist.github.com/archevel/48db152b85ad643bf33a
