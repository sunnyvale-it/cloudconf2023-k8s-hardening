# Implementing K8s multi-tenancy with Capsule

## Install Capsule

```console
$ helm repo add clastix https://clastix.github.io/charts
$ helm repo update
$ helm install capsule clastix/capsule -n capsule-system --create-namespace
```

## Your first tenant

```console
$ kubectl create -f - << EOF
apiVersion: capsule.clastix.io/v1beta2
kind: Tenant
metadata:
  name: sunnyvale
spec:
  owners:
  - name: denis
    kind: User
EOF
```

Check your tenant:

```console
$ kubectl get tenants
NAME        STATE    NAMESPACE QUOTA   NAMESPACE COUNT   NODE SELECTOR   AGE
sunnyvale   Active                     0                                 22s      
```

## Create users

Each tenant comes with a delegated user or group of users acting as the tenant admin. In the Capsule jargon, this is called the Tenant Owner. Other users can operate inside a tenant with different levels of permissions and authorizations assigned directly by the Tenant Owner.

[This script](https://github.com/clastix/capsule/blob/master/hack/create-user.sh) helps you set up a dummy kubeconfig for the **denis** user acting as owner of a tenant called **sunnyvale**

```console
$ curl https://raw.githubusercontent.com/clastix/capsule/master/hack/create-user.sh | bash -s denis sunnyvale
```

then login as tenant owner

```console
$ export KUBECONFIG=denis-sunnyvale.kubeconfig
```

Create namespaces

```console
$ kubectl create namespace sunnyvale-production
$ kubectl create namespace sunnyvale-development
```

## Limiting access to only the tenant's namespaces

Tenant Owners have full administrative permissions limited to only the namespaces in the assigned tenant. They can create any namespaced resource in their namespaces but they do not have access to cluster resources or resources belonging to other tenants they do not own:

```console
$ kubectl -n kube-system get pods
Error from server (Forbidden): pods is forbidden: User "denis" cannot list resource "pods" in API group "" in the namespace "kube-system"
```

More tutorials can be found [here](https://capsule.clastix.io/docs/general/tutorial).