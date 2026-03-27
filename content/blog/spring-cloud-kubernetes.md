---
title: "Demystifying and Integrating Spring Cloud Kubernetes"
date: 2020-04-24T00:00:00+05:30
draft: false
tags: ["spring-boot", "kubernetes", "cloud", "microservices"]
---

*Originally published on [Medium](https://medium.com/@vatsav.gs/demystifying-and-integrating-spring-cloud-kubernete-9679f391db76)*

With Spring Cloud being the most popular way of externalizing the configuration of any service running outside a kubernetes cluster. Now, Spring cloud kubernetes is becoming the new thing that helps in externalizing the config for the services running in a Kubernetes cluster leveraging the usage of ConfigMap in kubernetes.

Please refer to the link [https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) for a deeper understanding of config maps in kubernetes.

In this article we will try to understand how to integrate spring cloud kubernetes into a spring boot app and understand every part of the cloud kubernetes config file. Although, I am not covering the part of enabling the discovery client as part of this article. I assume, you have very basic understanding of kubernetes, kubectl commands and spring boot.

**NOTE**: I had a kubernetes cluster set up on aws. You can make use of minikube as a kubernetes cluster for your local machine.

## Getting Started

With addition of cloud kubernetes dependency in `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-config</artifactId>
    <version>1.1.2.RELEASE</version>
</dependency>
```

The Spring Cloud Kubernetes project makes Kubernetes ConfigMap's available during application bootstrapping and also triggers hot reloading of the context when changes are detected on ConfigMap's. The default behaviour is to create a ConfigMap which has metadata.name of either the name of your Spring application (as defined by `spring.application.name` property) or a custom name defined in the `bootstrap.yml` file under the following key `spring.cloud.kubernetes.config.name`.

I am considering to use `bootstrap.yml` for the sake of this article. The config of the bootstrap looks pretty much like this:

```yaml
spring:
  cloud:
    kubernetes:
      config:
        enabled: true # by default it is true. set false to disable.
        name: demo-api-config # config map name.
        sources:
          - name: demo-api-config
            namespace: demo # the namespace
```

This configuration enables Spring Cloud Kubernetes to read configuration from a ConfigMap named `demo-api-config` in the `demo` namespace.