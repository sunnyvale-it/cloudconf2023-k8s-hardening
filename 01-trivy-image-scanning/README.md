# Trivy image scanning



## Install Trivy (in my case on MacOS)

```console
$ brew install trivy  
```

More installation options at https://aquasecurity.github.io/trivy/v0.41/getting-started/installation/

## Scan an image from terminal

```console
$ trivy image sunnyvaleit/spring4shell-core:1.0
```

Trivy will start, download the necessary infos from the internet then scan the image producing this output (as the default it's in tabular format):

```console
...
│ org.springframework.boot:spring-boot (spring4shell.war)      │ CVE-2022-22965      │ CRITICAL │ 2.6.3             │ 2.5.12, 2.6.6                     │ spring-framework: RCE via Data Binding on JDK 9+             │
│                                                              │                     │          │                   │                                   │ https://avd.aquasec.com/nvd/cve-2022-22965   
...
````
