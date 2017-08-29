---
title: Anatomy of a calico-node container
---

`calico/node` can be regarded as a helper container that bundles together the
various components required for networking containers with Calico.  The key
components are:

-  Felix
-  BIRD
-  confd

In addition, we use runit for logging (`svlogd`) and init (`runsv`) services.

The [calicoctl repostiory](https://github.com/projectcalico/calicoctl) contains the Dockerfile for `calico/node` along with various
configuration files that are used to configure and "glue" these components
together.

<div class="alert alert-info" role="alert"><b>Note</b>: <samp>calico/node</samp> may be run in <b>policy only mode</b> in which Felix runs, but both BIRD and confd are removed. This provides policy management without route distribution between hosts. This mode can be enabled by setting the environment variable <samp>CALICO_NETWORKING=false</samp> before starting the node with <samp>calicoctl node run</samp>.</div>


#### Calico Felix agent

The Felix daemon is the heart of Calico networking.  Felix's primary job is to
program routes and ACL's on a workload host to provide desired connectivity to
and from workloads on the host.

Felix also programs interface information to the kernel for outgoing endpoint
traffic. Felix instructs the host to respond to ARPs for workloads with the
MAC address of the host.

For more details about Felix, please refer to the core [calico project](https://github.com/projectcalico/felix).

#### BIRD/BIRD6 internet routing daemon

BIRD is an open source BGP client that is used to exchange routing information
between hosts.  The routes that Felix programs into the kernel for endpoints
are picked up by BIRD and distributed to BGP peers on the network, which
provides inter-host routing.

There are two BIRD processes running in the `calico-node` container.  One for
IPv4 (bird) and one for IPv6 (bird6).

For more information on BIRD, please refer to the [BIRD internet routing daemon project](http://bird.network.cz/).

Calico uses a fork of the main BIRD repo, to include an additional feature
required for IPIP support when running Calico in a cloud environment.  Refer
to the [calico-bird repo](https://github.com/projectcalico/calico-bird) for more details.

#### confd templating engine

The confd templating engine monitors the etcd datastore for any changes to BGP
configuration (and some top level global default configuration such as AS
Number, logging levels, and IPAM information).

Confd dynamically generates BIRD configuration files based on the data in etcd,
triggered automatically from updates to the data.  When the configuration file
changes, confd triggers BIRD to load the new files.

For more information on confd, please refer to the [confd project](https://github.com/kelseyhightower/confd).

Calico uses a fork of the main confd repo which includes an additional change
to improve performance with the handling of watch prefixes
[calico-bird repo](https://github.com/projectcalico/calico-bird) for more details.
