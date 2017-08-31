---
title: Running Calico Node Container as a Service
---

This guide explains how to run Calico as a system process or service,
with a focus on running in a Dockerized deployment. We include
examples for Systemd, but the commands can be applied to other init
daemons such as upstart as well.

## Running the Calico Node Container as a Service
This section describes how to run the Calico node as a Docker container
in Systemd.  Included here is an EnvironmentFile that defines the Environment
variables for Calico and a sample systemd service file that uses the
environment file and starts the Calico node image as a service.

`calico.env` - the EnvironmentFile:

```shell
ETCD_ENDPOINTS=http://localhost:2379
ETCD_CA_FILE=""
ETCD_CERT_FILE=""
ETCD_KEY_FILE=""
CALICO_NODENAME=""
CALICO_NO_DEFAULT_POOLS=""
CALICO_IP=""
CALICO_IP6=""
CALICO_AS=""
CALICO_LIBNETWORK_ENABLED=true
CALICO_NETWORKING_BACKEND=bird
```

Be sure to update this environment file as necessary, such as modifying
ETCD_ENDPOINTS to point at the correct etcd cluster endpoints.

<div class="alert alert-info" role="alert"><b>Note</b>: The <samp>ETCD_CA_FILE</samp>, <samp>ETCD_CERT_FILE</samp>, and <samp>ETCD_KEY_FILE</samp> environment variables are required when using etcd with SSL/TLS. The values here are standard values for a non-SSL version of etcd, but you can use this template to define your SSL values if desired.
<p></p><p></p>
If <samp>CALICO_NODENAME</samp> is blank, the compute server hostname will be used to identify the Calico node.
<p></p><p></p>
If <samp>CALICO_IP</samp> or <samp>CALICO_IP6</samp> are left blank, Calico will use the currently configured values for the next hop IP addresses for this node&#8212;these can be configured through the node resource. If no next hop addresses have been configured, Calico will automatically determine an IPv4 next hop address by querying the host interfaces (and it will configure this value in the node resource). You may set <samp>CALICO_IP</samp> to <samp>autodetect</samp> to force auto-detection of IP address every time the node starts. If you set IP addresses through these environments it will reconfigure any values currently set through the node resource.
<p></p><p></p>
If <samp>CALICO_AS</samp> is left blank, Calico will use the currently configured value for the AS Number for the node BGP client&#8212;this can be configured through the node resource.  If no value is set,  Calico will inherit the AS Number from the global default value. If you set a value through this environment it will reconfigure any value currently set through the node resource.
<p></p><p></p>
The <samp>CALICO_NETWORKING_BACKEND</samp> defaults to use Bird as the routing daemon. This may also be set to gobgp (to use gobgp as the routing daemon, but note that this does not support IP in IP), or none (if routing is handled by an alternative mechanism).</div>


### Systemd Service Example

`calico-node.service` - the Systemd service:

```shell
[Unit]
Description=calico-node
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=/etc/calico/calico.env
ExecStartPre=-/usr/bin/docker rm -f calico-node
ExecStart=/usr/bin/docker run --net=host --privileged \
 --name=calico-node \
 -e NODENAME=${CALICO_NODENAME} \
 -e IP=${CALICO_IP} \
 -e IP6=${CALICO_IP6} \
 -e CALICO_NETWORKING_BACKEND=${CALICO_NETWORKING_BACKEND} \
 -e AS=${CALICO_AS} \
 -e NO_DEFAULT_POOLS=${CALICO_NO_DEFAULT_POOLS} \
 -e CALICO_LIBNETWORK_ENABLED=${CALICO_LIBNETWORK_ENABLED} \
 -e ETCD_ENDPOINTS=${ETCD_ENDPOINTS} \
 -e ETCD_CA_CERT_FILE=${ETCD_CA_CERT_FILE} \
 -e ETCD_CERT_FILE=${ETCD_CERT_FILE} \
 -e ETCD_KEY_FILE=${ETCD_KEY_FILE} \
 -v /var/log/calico:/var/log/calico \
 -v /run/docker/plugins:/run/docker/plugins \
 -v /lib/modules:/lib/modules \
 -v /var/run/calico:/var/run/calico \
 quay.io/calico/node:{{site.data.versions[page.version].first.title}}

ExecStop=-/usr/bin/docker stop calico-node

[Install]
WantedBy=multi-user.target
```

The Systemd service above does the following on start:
  - Confirm docker is installed under the `[Unit]` section
  - Get environment variables from the environment file above
  - Remove existing `calico-node` container (if it exists)
  - Start `calico/node`

The script will also stop the calico-node container when the service is stopped.

<div class="alert alert-info" role="alert"><b>Note</b>: Depending on how you've installed Docker, the name of the Docker service under the <samp>[Unit]</samp> section may be different (such as <samp>docker-engine.service</samp>). Be sure to check this before starting the service.</div>

