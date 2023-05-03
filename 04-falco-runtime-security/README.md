# Falco runtime security

## Prerequisites

This demo runs on a x86 based cluster, thus minikube running on Apple Silicon hardware is not expected to work.

## Deploy Falco on Kubernetes with Helm

```console
$ helm repo add falcosecurity https://falcosecurity.github.io/charts
$ helm repo update
$ helm install falco falcosecurity/falco -n falco --create-namespace --set driver.kind=modern-bpf
```

Wait a few seconds for the containers to start, then if everything went fine, you should see this in the Falco pod's log:

