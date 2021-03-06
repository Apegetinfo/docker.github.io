---
title: Troubleshoot swarm configurations
description: Learn how to troubleshoot your Docker Universal Control Plane cluster.
keywords: troubleshoot, etcd, rethinkdb, key, value, store, database, ucp, health, swarm
---

UCP automatically tries to heal itself by monitoring its internal
components and trying to bring them to a healthy state.

In most cases, if a single UCP component is in a failed state persistently,
you can restore the cluster to a healthy state by
removing the unhealthy node from the cluster and joining it again.
[Learn how to remove and join nodes](../configure/scale-your-cluster.md).

## Troubleshoot the etcd key-value store

UCP persists configuration data on an [etcd](https://coreos.com/etcd/)
key-value store and [RethinkDB](https://rethinkdb.com/) database that are
replicated on all manager nodes of the UCP swarm. These data stores are for
internal use only and should not be used by other applications.

### With the HTTP API

In this example we use `curl` for making requests to the key-value
store REST API, and `jq` to process the responses.

You can install these tools on a Ubuntu distribution by running:

```bash
$ sudo apt-get update && apt-get install curl jq
```

1. Use a client bundle to authenticate your requests.
[Learn more](../../user/access-ucp/cli-based-access.md).

2. Use the REST API to access the cluster configurations. The `$DOCKER_HOST`
   and `$DOCKER_CERT_PATH` environment variables are set when using the client
   bundle.

   ```bash
   $ export KV_URL="https://$(echo $DOCKER_HOST | cut -f3 -d/ | cut -f1 -d:):12379"

   $ curl -s \
        --cert ${DOCKER_CERT_PATH}/cert.pem \
        --key ${DOCKER_CERT_PATH}/key.pem \
        --cacert ${DOCKER_CERT_PATH}/ca.pem \
        ${KV_URL}/v2/keys | jq "."
   ```

To learn more about the key-value store REST API, check the
[etcd official documentation](https://coreos.com/etcd/docs/latest/).

### With the CLI client

The containers running the key-value store, include `etcdctl`, a command line
client for etcd. You can run it using the `docker exec` command.

The examples below assume you are logged in with ssh into a UCP manager node.

```bash
$ docker exec -it ucp-kv etcdctl \
        --endpoint https://127.0.0.1:2379 \
        --ca-file /etc/docker/ssl/ca.pem \
        --cert-file /etc/docker/ssl/cert.pem \
        --key-file /etc/docker/ssl/key.pem \
        cluster-health

member 16c9ae1872e8b1f0 is healthy: got healthy result from https://192.168.122.64:12379
member c5a24cfdb4263e72 is healthy: got healthy result from https://192.168.122.196:12379
member ca3c1bb18f1b30bf is healthy: got healthy result from https://192.168.122.223:12379
cluster is healthy
```

On failure, the command exits with an error code and no output.

To learn more about the `etcdctl` utility, check the
[etcd official documentation](https://coreos.com/etcd/docs/latest/).

## RethinkDB Database

User and organization data for Docker Enterprise Edition is stored in a
RethinkDB database which is replicated across all manager nodes in the UCP
swarm.

Replication and failover of this database is typically handled automatically by
UCP's own configuration management processes, but detailed database status and
manual reconfiguration of database replication is available through a command
line tool available as part of UCP.

The examples below assume you are logged in with ssh into a UCP manager node.

### Check the status of the database

{% raw %}
```bash
# NODE_ADDRESS will be the IP address of this Docker Swarm manager node
NODE_ADDRESS=$(docker info --format '{{.Swarm.NodeAddr}}')
# VERSION will be your most recent version of the docker/ucp-auth image
VERSION=$(docker image ls --format '{{.Tag}}' docker/ucp-auth | head -n 1)
# This command will output detailed status of all servers and database tables
# in the RethinkDB cluster.
docker container run --rm -v ucp-auth-store-certs:/tls docker/ucp-auth:${VERSION} --db-addr=${NODE_ADDRESS}:12383 db-status

Server Status: [
  {
    "ID": "ffa9cd5a-3370-4ccd-a21f-d7437c90e900",
    "Name": "ucp_auth_store_192_168_1_25",
    "Network": {
      "CanonicalAddresses": [
        {
          "Host": "192.168.1.25",
          "Port": 12384
        }
      ],
      "TimeConnected": "2017-07-14T17:21:44.198Z"
    }
  }
]
...
```
{% endraw %}

### Manually reconfigure database replication

{% raw %}
```bash
# NODE_ADDRESS will be the IP address of this Docker Swarm manager node
NODE_ADDRESS=$(docker info --format '{{.Swarm.NodeAddr}}')
# NUM_MANAGERS will be the current number of manager nodes in the cluster
NUM_MANAGERS=$(docker node ls --filter role=manager -q | wc -l)
# VERSION will be your most recent version of the docker/ucp-auth image
VERSION=$(docker image ls --format '{{.Tag}}' docker/ucp-auth | head -n 1)
# This reconfigure-db command will repair the RethinkDB cluster to have a
# number of replicas equal to the number of manager nodes in the cluster.
docker container run --rm -v ucp-auth-store-certs:/tls docker/ucp-auth:${VERSION} --db-addr=${NODE_ADDRESS}:12383 --debug reconfigure-db --num-replicas ${NUM_MANAGERS}

time="2017-07-14T20:46:09Z" level=debug msg="Connecting to db ..."
time="2017-07-14T20:46:09Z" level=debug msg="connecting to DB Addrs: [192.168.1.25:12383]"
time="2017-07-14T20:46:09Z" level=debug msg="Reconfiguring number of replicas to 1"
time="2017-07-14T20:46:09Z" level=debug msg="(00/16) Reconfiguring Table Replication..."
time="2017-07-14T20:46:09Z" level=debug msg="(01/16) Reconfigured Replication of Table \"grant_objects\""
...
```
{% endraw %}

> #### Loss of Quorum in RethinkDB Tables
>
> When there is loss of quorum in any of the RethinkDB tables, run the `reconfigure-db` command 
> with the `--emergency-repair` flag.

## Where to go next

* [Get support](../../get-support.md)
