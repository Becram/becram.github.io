---
title:  "Building Kubernetes Admission Webhooks (Part 1 of 2)"
last_modified_at: 2022-05-06T16:00:58-04:00
categories:
  - DevOps
tags:
  - Kubernetes
  - Golang
header:
  teaser: /assets/images/webhook-k8s-pt-1.jpeg

toc: true
toc_label: "What is here"
---



| SN |      Stack   | Technology        | 
|--| ------------- |:-------------:|
| 1 | Language     | Golang | 
| 2 |Orchestrator | Kubernetes(KinD)     |  

As you spend you more and more on the devops tools it wont take you long to realize that [Golang](https://go.dev/) is pretty powerful programming language given sheer volume of the tools build with it. Among those, the most prominent ones are the [Kubernetes](https://kubernetes.io), [Terraform](https://www.terraform.io/), [ArgoCD](https://argo-cd.readthedocs.io/en/stable/), [fluxCD](https://fluxcd.io/). As a SRE/DevOps Engineer I spend a-lot of time working on the internals of these tools profusely. It's just a matter of time that you get inspired to create a similar tools for automation purpose give the such amazing Open Source Community.

Kubernetes have taken the cloud eco-system by the storm. Kubernetes makes the complex system more visible and deployments much easier. Apart from this the extendibility of the Kubernetes API have made the cloud automation very robust.

I have been trying to understand Kubernetes API structure for a while, initially I tried to dive directly into the code from the officially provided [go-client](https://github.com/kubernetes/client-go), boy that was a painful to get grasp since its massive and novelty of structures. With massive number of nuts and bots it overwhelms you pretty fast. So before diving to code, I would suggest to understand the core data structure and concept first which has its own learning curve.

> Please the complete code for this series of tutorials at [https://github.com/Becram/kubernetes-webhook](https://github.com/Becram/kubernetes-webhook). I am a newbie to golang, and haven't written any tests

Before understanding what are Kubernetes Admission webhooks, let's first get to know basics. Here is quick summary of API resources, Kinds, and Objects. 
1. `Resource Type`:  loosely, an entity served by a Kubernetes API endpoint: pods, deployments, configmaps, etc.
2. `API Group`:  resource types are organized into versioned logical groups: apps/v1, batch/v1, storage.k8s.io/v1beta1, etc.
3. `Object`: a resource instance - every API endpoint deals with objects of a certain resource type.
4. `Kind`:  every object returned or accepted by the API must conform to an object schema - a certain composition of attributes defined by its kind: Pod, Deployment, ConfigMap, etc.

![](https://cdn-images-1.medium.com/max/1600/1*vPZJQHAlfv6YcJv1OI6eYQ.jpeg)

It's also important to differentiate between objects in a broad sense and Kubernetes `first-class` Objects - persistent entities like `Pod`, `Service`, or `Secret` serving as a record of intent for the cluster. While every API object must have an API version and kind attributes for the sake of its serialization and deserialization, not every API object is a `first-class` Kubernetes Object.

I would recommend to go through https://kubernetes.io/docs/reference/using-api/api-concepts/ for basic of Kubernetes apis.



#Understanding Go Modules

## **<span style="color:green"> Module k8s.io/api </span>**

Go is a statically typed programming language. So, where do all the structs corresponding to `Pods`, `ConfigMaps`, `Secrets`, and other first-class Kubernetes Objects live? Right, in [k8s.io/api](https://github.com/kubernetes/api).

```go
package main

import (
  "fmt"

  appsv1 "k8s.io/api/apps/v1"
  corev1 "k8s.io/api/core/v1"
)

func main() {
  deployment := appsv1.Deployment{
    Spec: appsv1.DeploymentSpec{
      Template: corev1.PodTemplateSpec{
        Spec: corev1.PodSpec{
          Containers: []corev1.Container{
            { Name:  "web", Image: "nginx:1.21" },
          },
        },
      },
    },
  }

  fmt.Printf("%#v", &deployment)
}
```

The module defines not only the top-level Kubernetes Objects like the `Deployment` above but also numerous auxiliary types for their inner attributes:

```go
// PodSpec is a description of a pod.
type PodSpec struct {
  Volumes []Volume `json:"volumes,omitempty" patchStrategy:"merge,retainKeys" patchMergeKey:"name" protobuf:"bytes,1,rep,name=volumes"`
  
  InitContainers []Container `json:"initContainers,omitempty" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,20,rep,name=initContainers"`
    
  Containers []Container `json:"containers" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,2,rep,name=containers"`

  EphemeralContainers []EphemeralContainer `json:"ephemeralContainers,omitempty" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,34,rep,name=ephemeralContainers"`

  RestartPolicy RestartPolicy `json:"restartPolicy,omitempty" protobuf:"bytes,3,opt,name=restartPolicy,casttype=RestartPolicy"`
    
  ...
}
```

## **<span style="color:green"> Module k8s.io/apimachinery </span>**

Unlike the narrowly-scoped `k8s.io/api` module, the [k8s.io/apimachinery](https://github.com/kubernetes/apimachinery) module is rather manifold. The [README](https://github.com/kubernetes/apimachinery/tree/3d7c63b4de4fdee1917284129969901d4777facc#purpose) describes its purpose as:

This library is a shared dependency for servers and clients to work with Kubernetes API infrastructure without direct type dependencies. Its first consumers are `k8s.io/kubernetes`, `k8s.io/client-go`, and `k8s.io/apiserver`.

It'd be hard to cover all the responsibilities of the apimachinery module in a single post without being too shallow. So i would recommend to read the official docs.

> Lets dive into the real project for admission webhooks in [part 2](https://bdhoju.com/devops/Building-Kubernetes-Admission-Webhooks-pt-2/)

### **<span style="color:green"> References </span>**

1. https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/
2. https://github.com/kubernetes/client-go
3. https://github.com/kubernetes/apimachinery
4. https://github.com/kubernetes/api
