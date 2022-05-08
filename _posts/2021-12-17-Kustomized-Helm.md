---
title:  "Kustomized Helm Structure"
last_modified_at: 2021-12-17T16:00:58-04:00
categories:
  - DevOps
tags:
  - Kubernetes
header:
  teaser: /assets/images/kustomized_helm.png

toc: true
toc_label: "What is here"
---



| SN |      Stack   | Technology        | 
|--| ------------- |:-------------:|
| 1 | Language     | yaml | 
| 2 | Templating | Helm/Kustomize    |  

As a devops/SRE maintaining and managing the yaml seems a cumbersome task. There is great a challenging in maintaining highly readable yaml while keeping it simple. [Helm](https://helm.sh/) and [Kustomize](https://kustomize.io/) are the two amazing tool to maintain these yaml. Both of these comes with amazing features choosing one over another is very difficult task in itself as the tradeoff is pretty high. So how about have both.


Another important challenge to maintain in multiple environment system to to maintain hierarchal yaml structure, such that the repeatitive values are maintained as base and values for each environment overlayed on top of it.

![](https://miro.medium.com/max/1160/1*i6fxGW9H-8V3iu-Xfh1ozQ.png)

As in above the single app (cert-manager in our case with jetstack helm chart) is broken in to two folders as `helm_base` and `overlays`, helm_base folder comprises of the helm chart definition in Chart.yaml file as

```go
apiVersion: v2
description: A Helm chart for cert-manager
name: cert-manager
type: application
version: v1.3.1
dependencies:
 — name: cert-manager
   version: v1.3.1
   repository: https://charts.jetstack.io
```


Note that the repository is defined as the dependencies this approach is helpful for defining multiple charts and segregating the values for it. In addition to the Chart definition it also comprises `kustomization.yml` this defines the resource definitions to be applied.

`helm_base/cert-manager/kustomization.yml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
commonLabels:
  app: cert-manager
resources:
- all.yml
```

You may notice the resources section which mentions the `all.yml` which does not exist we will get back to it later. commonLabels section adds this label to all the resources defined in `all.yml`.

Next folder is the overlay folder which contains the kustomize definition for each environment. The example above contains only for dev environment. For our `cert-manger chart` we have a folder for each environment. This folder contains kustomization definitions, helm overrride values and additional resources. Lets get to the `kustomization.yml` file.

`overlays/dev/cert-manager/kustomization.yml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../../helm_base/cert-manager
namespace: cert-manager
resources:
- namespace.yml
- cluster-issuer.yaml
```

When a this configuration is applied it goes back to the helm_base directory to particular app folder and applies the kustomization on that folder in addition to that there are other resources which needs to be applied in resources sections. Like I mentioned below the kustomization folder in base contains a kustomize config which contains the resource to apply named all.yml which doesnot exist currently. This `all.yml` file contains the resource definition which needs to be generated from the helm chart before we apply the overlay.

```sh
cd overlays/dev/cert-manager
helm dependency build ../../../helm_base/cert-manager
helm template ../../../helm_base/cert-manager --namespace cert-manager --name-template cert-manager -f  values-override.yaml   > ../../../helm_base/cert-manager/all.yml 
kubectl apply -k .
```

The last command applies all the resources in the all.yml and the additional resources defined in the resources section. Hence, we use the beautiful helm template as well as the powerful kustomize feature. Apart from this any resources generated from helm chart can be patch using kustomize which gives great power for changing the resources definition.

