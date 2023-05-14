# K0S in Docker Swarm (and Compose)

Currently kube-proxy and kube-router doesn't work properly with HA controller: https://github.com/k0sproject/k0s/issues/3120

Based on [k0s-in-docker](https://docs.k0sproject.io/v1.27.1+k0s.0/k0s-in-docker/).

Pro:
- Easier manage and rolling update of control components with docker swarm + automatic migration to other hosts in case of failure.

Const:
- Kubelet (k0s worker) should be started directly on host (without docker), at least because restart of kubelet in docker possibly will affect all pods inside it's docker container...
- You should setup and manage synching of data path or volumes storage between control nodes if you want live migration of k0s control containers between nodes.

Time track:
- [Filipp Frizzy](https://github.com/Friz-zy/): 

## Setup

### Docker Compose

1) generate secrets (only once per cluster).
This command populate `./secrets` directory near yaml files and exit with code 0
```
docker-compose -f generate-secrets.yml up
```

2) start controller
```
docker-compose -f controller.yml up -d
```

Wait untill all k0s containers will have status `(healthy)`
```
docker-compose -f controller.yml ps
```

create worker join token if you don't use static pregenerated one
`docker-compose -f controller.yml exec k0s-1 k0s token create --role worker > ./secrets/worker.token`

3) start kubelet
```
export HOSTNAME=$(hostname)
# for calico
modprobe ipt_ipvs xt_addrtype ip6_tables \
ip_tables nf_conntrack_netlink xt_u32 \
xt_icmp xt_multiport xt_set vfio-pci \
xt_bpf ipt_REJECT ipt_set xt_icmp6 \
xt_mark ip_set ipt_rpfilter \
xt_rpfilter xt_conntrack
docker-compose -f kubelet.yml up -d
```

### Docker Swarm

[Short intro into Swarm](https://gabrieltanner.org/blog/docker-swarm/).

## Known problems

### ETCD health status delay
- https://github.com/etcd-io/etcd/issues/2711
- https://github.com/etcd-io/etcd/issues/2340

ETCD cluster need some time (5 mins?) to detect hard powered off member.

### Scale k0s from 1 container to 3 and more

Single etcd node should be at least restarted with new parameters to become a cluster with two other nodes.

### worker kubeproxy & coredns work only with network 'bridge'

```
    #deploy:
      #mode: replicated
      #replicas: 3
    volumes:
      - /var/lib/k0s
      #- /var/lib/k0s/etcd
      #- /var/lib/k0s/containerd # for worker
    tmpfs:
      - /run
      - /var/run
    network_mode: "bridge"
```

### [k0s Manifest Deployer](https://docs.k0sproject.io/v1.27.1+k0s.0/manifests/) load manifests only from first level directories under `/var/lib/k0s/manifests`

You can check that controller imported all secrets for tokens: `k0s kubectl get secrets -A`.

### `Error: configmaps "worker-config-default-1.27" is forbidden: User "system:node:<NODENAME>" cannot get resource "configmaps" in API group "" in the namespace "kube-system": no relationship found between node '<NODENAME>' and this object`

Problem with [Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/).
I got this problem with pregenered worker token and fixed it by generating new join token after controller start.

### `error mounting "cgroup" to rootfs at "/sys/fs/cgroup"`

Don't mount `/sys:/sys:ro`

### Calico node failed with error `felix/table.go 1321: Failed to execute ip(6)tables-restore command error=exit status 2 errorOutput="iptables-restore v1.8.4 (legacy): Couldn't load match `rpfilter':No such file or directory\n\nError occurred at line: 10\nTry `iptables-restore -h' or 'iptables-restore --help' for more information.\n"`

Possibly `xt_rpfilter` kernel module is missing, you can check with this command in kubelet container:
`calicoctl node checksystem`

### Kube-proxy failed with some iptables error

Possibly some kernel modules missed or maybe kube-proxy version can't work with your system.
I got something like this with k0s v1.27.1-k0s.0 on old boot-to-docker vm with 4.19.130-boot2docker kernel.
However, looks like k0s v1.26.4-k0s.0 works fine.

https://www.tencentcloud.com/document/product/457/51209

## Useful commands
Controller containers contain tools like kubectl and etcdctl for managing cluster.
You can use it for debug or control with admin privileges:

### Get k0s cluster health status

`k0s kubectl get --raw='/livez?verbose'`

### Get all existed stuff in cluster

```
k0s kubectl get all -A
k0s kubectl get configmaps -A
k0s kubectl get secrets -A
k0s kubectl get nodes -A
k0s kubectl get namespaces -A
k0s kubectl get ingress -A
k0s kubectl get jobs -A
k0s kubectl get endpoints -A
k0s kubectl get users,roles,rolebindings,clusterroles -A
k0s kubectl get events -A
```

### Check k0s dynamic config (if enabled)
`k0s config status`

### Get member list of etcd cluster
```
etcdctl --endpoints=https://127.0.0.1:2379 \
--key=/var/lib/k0s/pki/etcd/server.key \
--cert=/var/lib/k0s/pki/etcd/server.crt \
--insecure-skip-tls-verify --write-out=table \
member list
```
or
`k0s etcd member-list`

### Get etcd member endpoint status and health

```
etcdctl --endpoints=https://127.0.0.1:2379 \
--key=/var/lib/k0s/pki/etcd/server.key \
--cert=/var/lib/k0s/pki/etcd/server.crt \
--insecure-skip-tls-verify --write-out=table \
endpoint status
```

```
etcdctl --endpoints=https://127.0.0.1:2379 \
--key=/var/lib/k0s/pki/etcd/server.key \
--cert=/var/lib/k0s/pki/etcd/server.crt \
--insecure-skip-tls-verify --write-out=table \
endpoint health
```
