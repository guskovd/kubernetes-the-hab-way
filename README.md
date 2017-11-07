# Kubernetes The Hab Way

A demo to show how Kubernetes can be set up with Habitat packages and services.

## Requirements

* Vagrant
* `cfssl` and `cfssljson`

## Setup

### Virtual machines

```
vagrant up
```

Verify all 3 Habitat supervisors are running and have joined the ring:

```
./scripts/list-peers
```

### Certificates

Create a CA and certificates for all components:

```
./scripts/generate-ssl-certificates
```

### etcd

First we start etcd:

```
# On node-0
sudo hab sup start schu/etcd --channel unstable --topology leader
# On node-1 and node-2
sudo hab sup start schu/etcd --channel unstable --topology leader --peer 192.168.222.10
```

NB: a service with topology `leader` requires at least 3 members. Only when 3
members are alive, the service will start. We take advantage of that to make
sure all three etcd member nodes start with the same "initial cluster" setting
(and therefore have the same cluster ID to be able to form a cluster).

Now we have to update the etcd.default service group configuration to make etcd
use our SSL certifcates (this has only to be done once for a service group; we
use node-0 here):

```
# On node-0
for f in /vagrant/certificates/{etcd.pem,etcd-key.pem,ca.pem}; do sudo hab file upload etcd.default 1 "${f}"; done
sudo hab config apply etcd.default 1 /vagrant/config/svc-etcd.toml
```

Finally we verify etcd is running:

```
# On node-0
export ETCDCTL_API=3
etcdctl --debug --insecure-transport=false --endpoints https://192.168.222.10:2379 --cacert /vagrant/certificates/ca.pem --cert /vagrant/certificates/etcd.pem --key /vagrant/certificates/etcd-key.pem member list
```

All 3 members should be started:

```
b82a52a6ff5c63c3, started, node-1, https://192.168.222.11:2380, https://192.168.222.11:2379
ddfa35d3c9a4c741, started, node-2, https://192.168.222.12:2380, https://192.168.222.12:2379
f1986a6cf0ad46aa, started, node-0, https://192.168.222.10:2380, https://192.168.222.10:2379
```
