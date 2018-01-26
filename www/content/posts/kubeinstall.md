---
title: "Set up a Kubernetes 1.9.1 cluster on Centos7"
date: 2018-01-26T14:54:24+01:00
---

# Pre-requisites

In order to set up a kubernetes cluster, follow steps below on all your
machines.

## Install kubernetes packages

Create files */etc/yum.repos.d/docker.repo* and */etc/yum.repos.d/kubernetes.repo*,
owned by root with read permission:

```
# Create in /etc/yum.repos.d/docker.repo
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
```

```
# Create /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://yum.kubernetes.io/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```

## Use yum to install:

```
docker-engine: 1.12.3-1.el7.centos  
kubeadm: 1.9.1-0  
kubectl: 1.9.1-0 
kubelet: 1.9.1-0 
kubernetes-cni: 0.6.0-0
```

## Set up systemd configuration

In */etc/systemd/system/kubelet.service.d/10-kubeadm.conf*, comment line below:

```
#Environment="KUBELET_CGROUP_ARGS=
```

Create file */etc/sysctl.d/90-kubernetes.conf*, owned by root with read
permission:

```
# Enable netfilter on bridges
# Required for weave (k8s v1.9.1) to start
net.bridge.bridge-nf-call-iptables = 1
```

Restart systemd

```
$ /bin/systemctl daemon-reload
$ /bin/systemctl enable docker 
$ /bin/systemctl enable kubelet
$ /bin/systemctl restart systemd-sysctl
```

# Create kubernetes cluster

On kubernetes master, as non-root user, run:

```bash
$ TOKEN=$(sudo -- kubeadm token generate)
# Add line below to access kubernetes API via ssh tunneling
$ SSH_TUNNEL_OPT="--apiserver-cert-extra-sans=localhost"
$ sudo -- kubeadm init $SSH_TUNNEL_OPT --token '$TOKEN'
$ mkdir -p $HOME/.kube
$ sudo cp /etc/kubernetes/admin.conf \$HOME/.kube/config
$ sudo chown -R qserv:qserv \$HOME/.kube
$ KUBEVER=\$(kubectl version | base64 | tr -d '\n')
# Install Weave network plugin
$ kubectl apply -f \"https://cloud.weave.works/k8s/net?k8s-version=\$KUBEVER\"
$ HASH=$(openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //')
```

On each other kubernetes machines,  as non-root user, run:

```
# Get $TOKEN and $HASH from master and export them on machine
# Assuming kubernetes master dns name is kubernetes-master.domain.net

$ kubeadm join --token '$TOKEN' --discovery-token-ca-cert-hash 'sha256:$HASH'
kubernetes-master.domain.net :6443
```

## Check cluster state

On kubernetes master, as non-root user, run:

```
$ kubectl get nodes
$ kubectl get pods
```
