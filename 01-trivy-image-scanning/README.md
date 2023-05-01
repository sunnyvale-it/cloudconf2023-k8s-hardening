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

Trivy will start, download the necessary infos from the internet then it will scan the image producing a similar output (as a default it's in tabular format):

```console
...
│ org.springframework.boot:spring-boot (spring4shell.war)      │ CVE-2022-22965      │ CRITICAL │ 2.6.3             │ 2.5.12, 2.6.6                     │ spring-framework: RCE via Data Binding on JDK 9+             │
│                                                              │                     │          │                   │                                   │ https://avd.aquasec.com/nvd/cve-2022-22965   
...
````

## Scan an image as part of your DevSecOps workflow 

Trivy is very useful if used within a DevOps pipeline, as it's able to interrupt the image publishing if one vulnerability with a specific severity is found. 

To check how Trivy is been used within a GitHub Action, please refer to [this](https://github.com/sunnyvale-it/CVE-2022-22965-PoC) repo.