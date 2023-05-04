# Falco runtime security

## Prerequisites

This demo runs on a x86 based cluster, thus K8s running on Apple Silicon hardware (aarch64) is not expected to work.

The following tools must be installed on the host:

- kubectl
- helm

Since Falco is strictly dependant from the host kernel (thus the K8s nodes OS version and kernel version), to reproduce this lab we setup a minikube cluster this way:

1) Host OS is Debian GNU/Linux 10 (buster) with kernel 4.19.0-23-amd64

2) Install Minikube (v1.30.1)

```console
$ curl -LO https://github.com/kubernetes/minikube/releases/download/v1.30.1/minikube-linux-amd64
$ sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

3) Install kvm2 driver

```console
$ curl -L https://github.com/kubernetes/minikube/releases/download/v0.30.0/docker-machine-driver-kvm2 -o /home/$(USER)/.minikube/bin/docker-machine-driver-kvm2
```

4) Start the Minikube cluster

```console
$ minikube start --driver=kvm2 --auto-update-drivers=false
```


## Deploy Falco on Kubernetes with Helm

```console
$ helm repo add falcosecurity https://falcosecurity.github.io/charts
$ helm repo update
$ helm install falco falcosecurity/falco -n falco --create-namespace --version 1.11.0
```

Wait a few seconds for the containers to start, then if everything went fine, you should see this in the Falco pod's log:


```console
$ kubectl logs falco-5mxvd -n falco
* Setting up /usr/src links from host
* Running falco-driver-loader for: falco version=0.28.0, driver version=5c0b863ddade7a45568c0ac97d037422c9efb750
* Running falco-driver-loader with: driver=module, compile=yes, download=yes
* Unloading falco module, if present
* Trying to load a system falco module, if present
* Success: falco module found and loaded with modprobe
Thu May  4 07:04:51 2023: Falco version 0.28.0 (driver version 5c0b863ddade7a45568c0ac97d037422c9efb750)
Thu May  4 07:04:51 2023: Falco initialized with configuration file /etc/falco/falco.yaml
Thu May  4 07:04:51 2023: Loading rules from file /etc/falco/falco_rules.yaml:
Thu May  4 07:04:51 2023: Loading rules from file /etc/falco/falco_rules.local.yaml:
Thu May  4 07:04:51 2023: Starting internal webserver, listening on port 8765
07:04:51.442418000: Notice Privileged container started (user=root user_loginuid=0 command=container:8ed271a4c832 k8s.ns=<NA> k8s.pod=<NA> container=8ed271a4c832 image=registry.k8s.io/pause:3.9) k8s.ns=<NA> k8s.pod=<NA> container=8ed271a4c832
07:04:51.459031000: Notice Privileged container started (user=root user_loginuid=0 command=container:4f949b2859a9 k8s.ns=kube-system k8s.pod=kube-proxy-qrrxc container=4f949b2859a9 image=registry.k8s.io/kube-proxy:v1.26.3) k8s.ns=kube-system k8s.pod=kube-proxy-qrrxc container=4f949b2859a9
07:04:51.474611000: Notice Privileged container started (user=<NA> user_loginuid=0 command=container:9cbe35746215 k8s.ns=<NA> k8s.pod=<NA> container=9cbe35746215 image=registry.k8s.io/pause:3.9) k8s.ns=<NA> k8s.pod=<NA> container=9cbe35746215
```

And the pod should be running without any restart

```console
$  kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
falco-5mxvd   1/1     Running   0          7m3s
```

