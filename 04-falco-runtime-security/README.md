# Falco runtime security

Falco is a tool by SysDig that helps you to answer the question "How do I know what is happening during runtime inside these containers?".

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

3) Manually install kvm2 driver (due to a bug in the latest release)

```console
$ curl -L https://github.com/kubernetes/minikube/releases/download/v0.30.0/docker-machine-driver-kvm2 -o /home/$USER/.minikube/bin/docker-machine-driver-kvm2
```

4) Start the Minikube cluster

```console
$ minikube start --driver=kvm2 --auto-update-drivers=false --memory 12000 --cpus 4
```

This is the Minikube VM we setup to run this demo

```console
$ minikube ssh
                         _             _            
            _         _ ( )           ( )           
  ___ ___  (_)  ___  (_)| |/')  _   _ | |_      __  
/' _ ` _ `\| |/' _ `\| || , <  ( ) ( )| '_`\  /'__`\
| ( ) ( ) || || ( ) || || |\`\ | (_) || |_) )(  ___/
(_) (_) (_)(_)(_) (_)(_)(_) (_)`\___/'(_,__/'`\____)

$ cat /etc/os-release 
NAME=Buildroot
VERSION=2021.02.12-1-gf97079d-dirty
ID=buildroot
VERSION_ID=2021.02.12
PRETTY_NAME="Buildroot 2021.02.12"

$ uname -r
5.10.57
```


## Deploy Falco on Kubernetes with Helm

```console
$ helm repo add falcosecurity https://falcosecurity.github.io/charts
$ helm repo update
$ helm upgrade falco falcosecurity/falco --values values.yaml -n falco --create-namespace --version 3.1.4 --install
```

Wait a few seconds for the containers to start, then if everything went fine, you should see this in the Falco pod's log:


```console
$ kubectl logs falco-5mxvd -n falco
Defaulted container "falco" out of: falco, falcoctl-artifact-follow, falcoctl-artifact-install (init)
Thu May  4 09:56:34 2023: Falco version: 0.34.1 (x86_64)
Thu May  4 09:56:34 2023: Falco initialized with configuration file: /etc/falco/falco.yaml
Thu May  4 09:56:34 2023: Loading rules from file /etc/falco/falco_rules.yaml
Thu May  4 09:56:34 2023: The chosen syscall buffer dimension is: 8388608 bytes (8 MBs)
Thu May  4 09:56:34 2023: Starting health webserver with threadiness 2, listening on port 8765
Thu May  4 09:56:34 2023: Enabled event sources: syscall
Thu May  4 09:56:34 2023: Opening capture with modern BPF probe.
Thu May  4 09:56:34 2023: One ring buffer every '2' CPUs.
```

And the pod should be running without any restart

```console
$  kubectl get pods -n falco
NAME          READY   STATUS    RESTARTS   AGE
falco-xqzn2   2/2     Running   0          58s
```

## Putting Falco to the test

Now that Falco is deployed and running, let’s see if it can detect some suspicious activity.

As containers are not supposed to be interactively accessed, one of the biggest red flags of a compromised container is when a terminal shell spawns.

```console
$ kubectl run -i --rm --tty busybox --image=busybox --restart=Never -n default -- /bin/sh
```

In another terminal, if you look in the Falco logs you should see something like:

```console
$ kubectl logs falco-5mxvd 
...
07:28:29.510366389: Notice A shell was spawned in a container with an attached terminal (user=root user_loginuid=-1 k8s.ns=<NA> k8s.pod=<NA> container=4946bb8f2dfb shell=sh parent=<NA> cmdline=sh terminal=34816 container_id=4946bb8f2dfb image=<NA>) k8s.ns=<NA> k8s.pod=<NA> container=4946bb8f2dfb
```

The container id 4946bb8f2dfb is the one inside our busybox pod:

```console
$ kubectl describe pod busybox -n default | grep 4946bb8f2dfb
    Container ID:  docker://4946bb8f2dfb326bf4054ca1463ca71503818e84a8b1394ed9977f01669c74ea
```

If you want to generate different event, try the Falco Event Generator:

```console
$ helm install event-generator falcosecurity/event-generator \
  --namespace event-generator \
  --create-namespace \
  --set config.loop=false \
  --set config.actions=""
```

If you inspect the Falco logs, now you can see that all the event are trapped and logged:

```console
$ kubectl logs falco-cqt4x -n falco -f
...
06:45:21.243469810: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/2803989181 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.249329593: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/1997477483 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.255589551: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/2727704494 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.260553378: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/discovery/10.96.0.1_443/servergroups.json.828087719 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.263663379: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/84358683 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.272079629: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/1741022107 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.277187228: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/discovery/10.96.0.1_443/policy/v1/serverresources.json.3760562882 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.277875744: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/1319823106 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.287655829: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/555139102 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.294072549: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/3972841148 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.300180049: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/1078191325 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.300647646: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/discovery/10.96.0.1_443/networking.k8s.io/v1/serverresources.json.100341313 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.311016263: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/2062572944 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.320267193: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/3151358547 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.325576024: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/discovery/10.96.0.1_443/apps/v1/serverresources.json.3385454730 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.337371811: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/2992096203 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.344727244: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/581842572 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.352134673: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/4185323712 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.357969911: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/2386553025 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.365557849: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/1202177541 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
06:45:21.373296051: Error File below / or /root opened for writing (user=root user_loginuid=-1 command=event-generator run pid=3795 parent=event-generator file=/root/.kube/cache/http/.diskv-temp/582415147 program=event-generator container_id=2826788a2163 image=falcosecurity/event-generator) k8s.ns=event-generator k8s.pod=event-generator-sn5vp container=2826788a2163
```

