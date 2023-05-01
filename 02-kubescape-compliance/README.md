# Kubescape compliance/misconfiguration scanning

## Install Kuberscape

```console
$ curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash
```

## Scan the cluster

Your first scan

```console
$ kubescape scan --enable-host-scan --verbose
```

The output ends with a summary table as the following:

```
Controls: 65 (Failed: 14, Passed: 43, Action Required: 8)
Failed Resources by Severity: Critical — 0, High — 2, Medium — 20, Low — 7

+----------+-------------------------------------------------------+------------------+---------------+--------------------+
| SEVERITY |                     CONTROL NAME                      | FAILED RESOURCES | ALL RESOURCES |    % RISK-SCORE    |
+----------+-------------------------------------------------------+------------------+---------------+--------------------+
| Critical | API server insecure port is enabled                   |        0         |       1       |         0%         |
| Critical | Disable anonymous access to Kubelet service           |        0         |       1       |         0%         |
| Critical | Enforce Kubelet client TLS authentication             |        0         |       2       |         0%         |
| Critical | CVE-2022-39328-grafana-auth-bypass                    |        0         |       0       |         0%         |
| High     | Forbidden Container Registries                        |        0         |      10       | Action Required ** |
| High     | Resources memory limit and request                    |        0         |      10       | Action Required ** |
| High     | Resource limits                                       |        2         |      10       |        20%         |
| High     | Applications credentials in configuration files       |        0         |      24       | Action Required ** |
| High     | List Kubernetes secrets                               |        0         |      72       |         0%         |
| High     | Host PID/IPC privileges                               |        0         |      10       |         0%         |
| High     | HostNetwork access                                    |        0         |      10       |         0%         |
| High     | Writable hostPath mount                               |        0         |      10       |         0%         |
| High     | Insecure capabilities                                 |        0         |      10       |         0%         |
| High     | HostPath mount                                        |        0         |      10       |         0%         |
| High     | Resources CPU limit and request                       |        0         |      10       | Action Required ** |
| High     | Instance Metadata API                                 |        0         |       0       |         0%         |
| High     | Privileged container                                  |        0         |      10       |         0%         |
| High     | CVE-2021-25742-nginx-ingress-snippet-annotation-vu... |        0         |       0       |         0%         |
| High     | Workloads with Critical vulnerabilities exposed to... |        0         |       0       | Action Required *  |
| High     | Workloads with RCE vulnerabilities exposed to exte... |        0         |       0       | Action Required *  |
| High     | CVE-2022-23648-containerd-fs-escape                   |        0         |       1       |         0%         |
| High     | RBAC enabled                                          |        0         |       1       |         0%         |
| High     | CVE-2022-47633-kyverno-signature-bypass               |        0         |       0       |         0%         |
| Medium   | Exec into container                                   |        0         |      72       |         0%         |
```

## Pluggable  frameworks

Kubescape support the use of different scanning frameworks:

```console
$ kubescape list frameworks      
+----------------------+
| SUPPORTED FRAMEWORKS |
+----------------------+
|     AllControls      |
+----------------------+
|       ArmoBest       |
+----------------------+
|      DevOpsBest      |
+----------------------+
|        MITRE         |
+----------------------+
|         NSA          |
+----------------------+
|    cis-aks-t1.2.0    |
+----------------------+
|    cis-eks-t1.2.0    |
+----------------------+
|   cis-v1.23-t1.0.1   |
+----------------------+
```

Scan with a different/specific framework, for example

```console
$ kubescape scan framework nsa --enable-host-scan --verbose
```