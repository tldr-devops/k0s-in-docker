# K0S in Docker Swarm (and Compose)

[![#StandWithBelarus](https://img.shields.io/badge/Belarus-red?label=%23%20Stand%20With&labelColor=white&color=red)
<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/e/ea/Presidential_Standard_of_Belarus_%28fictional%29.svg/240px-Presidential_Standard_of_Belarus_%28fictional%29.svg.png" width="20" height="20" alt="Voices From Belarus" />](https://bysol.org/en/) [![Stand With Ukraine](https://raw.githubusercontent.com/vshymanskyy/StandWithUkraine/main/badges/StandWithUkraine.svg)](https://vshymanskyy.github.io/StandWithUkraine)

Based on [k0s-in-docker](https://docs.k0sproject.io/v1.27.1+k0s.0/k0s-in-docker/).

Status: experimental. Currently kube-proxy (kubelet component) doesn't work on hosts with docker swarm possibly because of iptables conflict.

Pro:
- Easier management and rolling updates of control components with Docker Swarm, including automatic migration to other hosts in case of failure.

Const:
- Better would be start kubelet (k0s worker) directly on the host, without Docker. This is important because restarting kubelet within Docker could potentially impact all the pods running inside the Docker container.
- If you want to achieve live migration of k0s control containers between nodes, you need to set up and manage data path or volume storage synchronization between control nodes.

Alternatively, you can consider running Kubernetes within Kubernetes using the following projects:
- [K0smotron](https://github.com/k0sproject/k0smotron)
- [Kubernetes-in-Kubernetes](https://github.com/kubefarm/kubernetes-in-kubernetes)

Time track:
- [Filipp Frizzy](https://github.com/Friz-zy/): 56h 50m

### About the Author

Hello, everyone! My name is Filipp, and I have been working with high load distribution systems and services, security, monitoring, continuous deployment and release management (DevOps domain) since 2012.

One of my passions is developing DevOps solutions and contributing to the open-source community. By sharing my knowledge and experiences, I strive to save time for both myself and others while fostering a culture of collaboration and learning.

I had to leave my home country, Belarus, due to my participation in [protests against the oppressive regime of dictator Lukashenko](https://en.wikipedia.org/wiki/2020%E2%80%932021_Belarusian_protests), who maintains a close affiliation with Putin. Since then, I'm trying to build my life from zero in other countries.

If you are seeking a skilled DevOps lead or architect to enhance your project, I invite you to connect with me on [LinkedIn](https://www.linkedin.com/in/filipp-frizzy-289a0360/) or explore my valuable contributions on [GitHub](https://github.com/Friz-zy/). Let's collaborate and create some cool solutions together :)

### How You Can Support My Projects

There are a couple of ways you can support my projects:

* **Sending Pull Requests (PRs)**:  
    If you come across any improvements or suggestions for my configurations or texts, feel free to send me Pull Requests (PRs) with your proposed changes. I appreciate your contributions <3

* **Making Donations**:  
    If you find my projects valuable and would like to support them financially, you can make a donation. Your contributions will go towards further development, maintenance, and improvement of the projects. Your support is greatly appreciated and helps to ensure the continued success of the projects.

  - [![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/filipp_frizzy)
  - [donationalerts.com/r/filipp_frizzy](https://www.donationalerts.com/r/filipp_frizzy)
  - ETH 0xCD9fC1719b9E174E911f343CA2B391060F931ff7
  - BTC bc1q8fhsj24f5ncv3995zk9v3jhwwmscecc6w0tdw3

Thank you for considering supporting my work. Your involvement and contributions make a significant difference in the growth and success of my projects.

## Setup

### Docker Compose

1) generate secrets (only once per cluster).
This command populate `./secrets` directory near yaml files and exit with code 0
```
docker-compose -f generate-secrets.yml up
docker-compose -f generate-secrets.yml down
echo "externalAddress=$(hostname -i)" >> .env
```

2) start controller
```
docker-compose -f controller.yml up -d
```

Wait untill all k0s containers up and running
```
docker-compose -f controller.yml ps
docker-compose -f controller.yml exec k0s-1 k0s kubectl get --raw='/livez?verbose'
```

Optional create worker join token if you don't use static pregenerated one
`docker-compose -f controller.yml exec k0s-1 k0s token create --role worker > ./secrets/worker.token`

3) start kubelet

Load necessary kernel modules for Calico if you use it
```
modprobe ipt_ipvs xt_addrtype ip6_tables \
ip_tables nf_conntrack_netlink xt_u32 \
xt_icmp xt_multiport xt_set vfio-pci \
xt_bpf ipt_REJECT ipt_set xt_icmp6 \
xt_mark ip_set ipt_rpfilter \
xt_rpfilter xt_conntrack
```

Start Kubelet
```
docker-compose -f kubelet.yml up -d
```

### Docker Swarm

* [Short intro into Swarm](https://gabrieltanner.org/blog/docker-swarm/)
* [Extended tutorial](https://dockerswarm.rocks/)

1) generate secrets (only once per cluster).
This command populate `./secrets` directory near yaml files and exit with code 0
```
docker-compose -f generate-secrets.yml up
docker-compose -f generate-secrets.yml down
echo "externalAddress=$(hostname -i)" >> .env
```

2) [setup docker swarm](https://docs.docker.com/engine/reference/commandline/swarm_init/)
```
docker swarm init --advertise-addr $(hostname -i)
```

3) start controller
```
export $(grep -v '^#' .env | xargs -d '\n')
docker stack deploy --compose-file controller.yml k0s
```

Wait untill all k0s containers up and running
```
docker stack ps k0s
docker service ls
```

4) start kubelet

Load necessary kernel modules for Calico if you use it
```
modprobe ipt_ipvs xt_addrtype ip6_tables \
ip_tables nf_conntrack_netlink xt_u32 \
xt_icmp xt_multiport xt_set vfio-pci \
xt_bpf ipt_REJECT ipt_set xt_icmp6 \
xt_mark ip_set ipt_rpfilter \
xt_rpfilter xt_conntrack
```

Docker Swarm doesn't support privileged mode so run it with Compose
```
docker-compose -f kubelet.yml up -d
```

## Known problems

### * ETCD health status delay
- https://github.com/etcd-io/etcd/issues/2711
- https://github.com/etcd-io/etcd/issues/2340

ETCD cluster need some time (5 mins?) to detect hard powered off member.

### * Scale k0s from 1 container to 3 and more

Single etcd node should be at least restarted with new parameters to become a cluster with two other nodes.

### * Docker Swarm doesn't resolve IP of container into DNS name
Docker Swarm doesn't resolve IP of container into DNS name while container health is not `healthy`.
As we need DNS resolution for getting containers up, I disabled healthcheck =(

### * ETCD in Docker Swarm produce `remote error: tls: bad certificate`
ETCD peer certificate contain only one ip while in swarm container name resolved into other service ip.
Issue: https://github.com/k0sproject/k0s/issues/3318. Solution: use `deploy.endpoint_mode: dnsrr` config.

### * Worker kubeproxy & coredns work only with network 'bridge' or 'host'

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

### * [k0s Manifest Deployer](https://docs.k0sproject.io/v1.27.1+k0s.0/manifests/) load manifests only from first level directories under `/var/lib/k0s/manifests`

You can check that controller imported all secrets for tokens: `k0s kubectl get secrets -A`.

### * `Error: configmaps "worker-config-default-1.27" is forbidden: User "system:node:<NODENAME>" cannot get resource "configmaps" in API group "" in the namespace "kube-system": no relationship found between node '<NODENAME>' and this object`

Problem with [Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/).
I got this problem with pregenered worker token and fixed it by generating new join token after controller start.

### * `error mounting "cgroup" to rootfs at "/sys/fs/cgroup"`

Don't mount `/sys:/sys:ro`

### * Calico node failed with error `felix/table.go 1321: Failed to execute ip(6)tables-restore command error=exit status 2 errorOutput="iptables-restore v1.8.4 (legacy): Couldn't load match `rpfilter':No such file or directory\n\nError occurred at line: 10\nTry `iptables-restore -h' or 'iptables-restore --help' for more information.\n"`

Possibly `xt_rpfilter` kernel module is missing, you can check with this command in kubelet container:
`calicoctl node checksystem`

### * Kube-proxy failed with some iptables error

Possibly some kernel modules missed or maybe kube-proxy version can't work with your system.
I got something like this with k0s v1.27.1-k0s.0 on old boot-to-docker vm with 4.19.130-boot2docker kernel.
However, looks like k0s v1.26.4-k0s.0 works fine.

https://www.tencentcloud.com/document/product/457/51209

## Useful commands
Controller containers contain tools like kubectl and etcdctl for managing cluster.
You can use it for debug or control with admin privileges:

### * Get k0s cluster health status

`k0s kubectl get --raw='/livez?verbose'`

### * Get all existed stuff in cluster

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

### * Check k0s dynamic config (if enabled)
`k0s config status`

### * Get member list of etcd cluster
```
etcdctl --endpoints=https://127.0.0.1:2379 \
--key=/var/lib/k0s/pki/etcd/server.key \
--cert=/var/lib/k0s/pki/etcd/server.crt \
--insecure-skip-tls-verify --write-out=table \
member list
```
or
`k0s etcd member-list`

### * Get etcd member endpoint status and health

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
