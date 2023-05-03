# Validate resources with Kyverno

Kyverno is a policy engine designed for Kubernetes

Kyverno policies can validate, mutate, generate, and cleanup Kubernetes resources plus ensure OCI image supply chain security.

## Kyverno installation

Kyverno installation using the Helm chart.

```console
$ helm repo add kyverno https://kyverno.github.io/kyverno/  
$ helm repo update
$ helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

if everything went fine:

```
NAME: kyverno
LAST DEPLOYED: Tue May  2 11:43:23 2023
NAMESPACE: kyverno
STATUS: deployed
REVISION: 1
NOTES:
Chart version: 2.7.2
Kyverno version: v1.9.2

Thank you for installing kyverno! Your release is named kyverno.
```

and

```console
$ kubectl get deployment -n kyverno
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
kyverno                      1/1     1            1           98s
kyverno-cleanup-controller   1/1     1            1           98s
```

## Apply policies

The [Kyverno documentation](https://kyverno.io/policies/?ref=anaisurl.com&policytypes=Pod) provides a vast list of Kubernetes policies that you can deploy to Kyverno as YAML manifests.

For example, let's try to disallow the `latest` tag on container images:

```console
$ cat <<'EOF' | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
  annotations:
    policies.kyverno.io/title: Disallow Latest Tag
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/minversion: 1.6.0
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod
    policies.kyverno.io/description: >-
      The ':latest' tag is mutable and can lead to unexpected errors if the
      image changes. A best practice is to use an immutable tag that maps to
      a specific version of an application Pod. This policy validates that the image
      specifies a tag and that it is not called `latest`.      
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: require-image-tag
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "An image tag is required."
      pattern:
        spec:
          containers:
          - image: "*:*"
  - name: validate-image-tag
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Using a mutable image tag e.g. 'latest' is not allowed."
      pattern:
        spec:
          containers:
          - image: "!*:latest"
EOF
```

Then, if we try to apply a manifest containing a `latest` image:

```console
$ cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
EOF
```

we should get

```
Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request: 

policy Deployment/default/nginx-deployment for resource violation: 

disallow-latest-tag:
  autogen-validate-image-tag: 'validation error: Using a mutable image tag e.g. ''latest''
    is not allowed. rule autogen-validate-image-tag failed at path /spec/template/spec/containers/0/image/'
```

You can use a policy to ensure no image with a know vulnerability is installed on the cluster:

```console
$ cat <<'EOF' | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image-vulns-cve-2022-42889
  annotations:
    policies.kyverno.io/title: Verify Image Check CVE-2022-42889
    policies.kyverno.io/category: Software Supply Chain Security
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod
    policies.kyverno.io/minversion: 1.8.3
    kyverno.io/kyverno-version: 1.9.0
    kyverno.io/kubernetes-version: "1.24"
    policies.kyverno.io/description: >-
      CVE-2022-42889 is a critical vulnerability in the Apache Commons Text library which
      could lead to arbitrary code executions and occurs in versions 1.5 through 1.9. Detecting
      the affected package may be done in an SBOM by identifying the "commons-text" package
      with one of the affected versions. This policy checks attested SBOMs in CycloneDX format of an image
      specified under `imageReferences` and denies it if it contains versions 1.5-1.9 of the commons-text
      package. Using this for your own purposes will require customizing the `imageReferences`,
      `subject`, and `issuer` fields based on your image signatures and attestations.      
spec:
  validationFailureAction: Enforce
  webhookTimeoutSeconds: 10
  rules:
    - name: cve-2022-42889
      match:
        any:
        - resources:
            kinds:
              - Pod
      verifyImages:
      - imageReferences:
        - "docker.io/sunnyvaleit/text4shell*"
        attestations:
        - predicateType: https://cyclonedx.org/schema
          attestors:
          - entries:
            - keyless:
                subject: "mysubject"
                issuer: "myissuer"
                rekor:
                  url: https://rekor.sigstore.dev
          conditions:
          - all:
            - key: "{{ components[?name=='commons-text'].version || 'none' }}"
              operator: AllNotIn
              value: ["1.5","1.6","1.7","1.8","1.9"]
EOF
```

and if we try to run a Deployment referring to the image specified in the policy:


```console
$ cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: text4shell-deployment
  labels:
    app: text4shell
spec:
  replicas: 2
  selector:
    matchLabels:
      app: text4shell
  template:
    metadata:
      labels:
        app: text4shell
    spec:
      containers:
      - name: text4shell
        image: docker.io/sunnyvaleit/text4shell:1.0
        ports:
        - containerPort: 8080
EOF
```

We get the error that the image can not be used since there's no attestation of authenticity/trust on Rekor log service.

```
Error from server: error when creating "STDIN": admission webhook "mutate.kyverno.svc-fail" denied the request: 

policy Deployment/default/text4shell-deployment for resource violation: 

check-image-vulns-cve-2022-42889:
  autogen-cve-2022-42889: |
    failed to verify image docker.io/sunnyvaleit/text4shell:1.0: no matching attestations:
```

With policies you can validate, mutate or generate resources on the cluster. More at https://kyverno.io


