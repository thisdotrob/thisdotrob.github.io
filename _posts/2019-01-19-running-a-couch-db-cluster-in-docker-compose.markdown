---
layout: post
title:  "Running a couchDB cluster with docker-compose"
date:   2019-01-19 15:20:00
categories: couch databases docker
---

There is documentation out there explaining how to run couchDB as a docker container
but info on how to run a cluster in docker-compose is scarce.

This post documents the steps you need to take to get a 3 node couch DB cluster
running in docker-compose.

Why would you want to do this? You probably wouldn't want to in production. But
for local development, testing and figuring out how couch works it is really useful
to be able to spin up a cluster in a single command, on a single host.

Step 1: create a `docker-compose.yml` file with simple definitions for the 3 nodes
in the cluster:
```yml
version: "3.7"

services:
  couchdb0:
    image: couchdb:2.3.0

  couchdb1:
    image: couchdb:2.3.0

  couchdb2:
    image: couchdb:2.3.0
```

Step 2: create a bootstrap script to join the nodes, using `couchdb0` as the main
setup node (this doesn't mean it is a master mode once the cluster running, couch
has no such concept):
```sh
# Bind the clustered interface to all IP addresses availble on this machine:
curl -X PUT http://couchdb0:5984/_node/_local/_config/chttpd/bind_address -d '"0.0.0.0"'
curl -X PUT http://couchdb1:5984/_node/_local/_config/chttpd/bind_address -d '"0.0.0.0"'
curl -X PUT http://couchdb2:5984/_node/_local/_config/chttpd/bind_address -d '"0.0.0.0"'

# Set the uuid for each node to be the same:
curl -X PUT http://couchdb0:5984/_node/_local/_config/couchdb/uuid -d '"d8702bee1e72f1517b27ee0da3000608"'
curl -X PUT http://couchdb1:5984/_node/_local/_config/couchdb/uuid -d '"d8702bee1e72f1517b27ee0da3000608"'
curl -X PUT http://couchdb2:5984/_node/_local/_config/couchdb/uuid -d '"d8702bee1e72f1517b27ee0da3000608"'

# Set the shared http secret for cookie creation:
curl -X PUT http://couchdb0:5984/_node/_local/_config/couch_httpd_auth/secret -d '"d8702bee1e72f1517b27ee0da30011c3"'
curl -X PUT http://couchdb1:5984/_node/_local/_config/couch_httpd_auth/secret -d '"d8702bee1e72f1517b27ee0da30011c3"'
curl -X PUT http://couchdb2:5984/_node/_local/_config/couch_httpd_auth/secret -d '"d8702bee1e72f1517b27ee0da30011c3"'

# Create the admin user and password:
curl -X PUT http://couchdb0:5984/_node/_local/_config/admins/admin -d '"password"'
curl -X PUT http://couchdb1:5984/_node/_local/_config/admins/admin -d '"password"'
curl -X PUT http://couchdb2:5984/_node/_local/_config/admins/admin -d '"password"'

# Use the main setup node to join the nodes together:
curl -X POST -H "Content-Type: application/json" http://admin:password@couchdb0:5984/_cluster_setup \
        -d '{"action": "enable_cluster", "bind_address":"0.0.0.0", "username": "admin", "password":"password", "port": 5984, "node_count": "3", "remote_node": "couchdb1.docker.com", "remote_current_user": "admin", "remote_current_password": "password" }'
curl -X POST -H "Content-Type: application/json" http://admin:password@couchdb0:5984/_cluster_setup \
        -d '{"action": "add_node", "host":"couchdb1.docker.com", "port": 5984, "username": "admin", "password":"password"}'
curl -X POST -H "Content-Type: application/json" http://admin:password@couchdb0:5984/_cluster_setup \
        -d '{"action": "enable_cluster", "bind_address":"0.0.0.0", "username": "admin", "password":"password", "port": 5984, "node_count": "3", "remote_node": "couchdb2.docker.com", "remote_current_user": "admin", "remote_current_password": "password" }'
curl -X POST -H "Content-Type: application/json" http://admin:password@couchdb0:5984/_cluster_setup \
        -d '{"action": "add_node", "host":"couchdb2.docker.com", "port": 5984, "username": "admin", "password":"password"}'

echo Finish cluster setup and add system databases:
curl -X POST -H "Content-Type: application/json" http://admin:password@couchdb0:5984/_cluster_setup -d '{"action": "finish_cluster"}'
```

```yml
version: "3.7"

services:
  bootstrap-cluster:
    image: node:11.6.0-alpine
    networks:
      couch:
        aliases:
          - bootstrapper.gower.st
    volumes:
      - ./scripts:/scripts
    command: sh -c "apk add --no-cache curl && /scripts/bootstrap_cluster.sh

  loadbalancer:
    image: haproxy:1.9.1
    ports:
      - 5984:5984
    networks:
      couch:
        aliases:
          - couchdb.docker.com
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    command: bash -c "haproxy -f /usr/local/etc/haproxy/haproxy.cfg"

  couchdb0:
    environment:
      NODENAME: couchdb0.docker.com
    image: couchdb:2.3.0
    ports:
      - 15984:5984
    networks:
      couch:
        aliases:
          - couchdb0.docker.com

  couchdb1:
    environment:
      NODENAME: couchdb1.docker.com
    image: couchdb:2.3.0
    networks:
      couch:
        aliases:
          - couchdb1.docker.com

  couchdb2:
    environment:
      NODENAME: couchdb2.docker.com
    image: couchdb:2.3.0
    networks:
      couch:
        aliases:
          - couchdb2.docker.com

networks:
  couch:
```
