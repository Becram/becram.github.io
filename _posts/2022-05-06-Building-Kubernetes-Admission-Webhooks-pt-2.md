---
title:  "Building Kubernetes Admission Webhooks (Part 2 of 2)"
last_modified_at: 2022-05-07T16:00:58-04:00
categories:
  - DevOps
tags:
  - Kubernetes
  - Golang
header:
  teaser: /assets/images/webhook-k8s-pt-2.jpeg

toc: true
toc_label: "What is here"
---



| SN |      Stack   | Technology        | 
|--| ------------- |:-------------:|
| 1 | Language     | Golang | 
| 2 |Orchestrator | Kubernetes(KinD)     |  

This is 2 part series so you ight want to visit part-1 [here](https://bdhoju.com/devops/Building-Kubernetes-Admission-Webhooks-pt-1/) 

If you have used Kubernetes for a while it is not hard to notice that most services use admission webhooks profusely. For instance, if you use [nginx-ingress-controller](https://kubernetes.github.io/ingress-nginx/) you will notice it deploys a webhook to detect the ingress resources and modify the `Nginx configuration` based on the resource annotations. Hence, admission webhooks are a pretty powerful feature in the Kubernetes. With the extendibility of the API in Kubernetes, users can create their webhooks to modify/validate the resources before they are applied to the cluster. This becomes very handy to maintain the sanity in your cluster.

From [offical docs](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)

>What are admission webhooks? <br>
Admission webhooks are HTTP callbacks that receive admission requests and do something with them. You can define two types of admission webhooks, validating admission webhook and mutating admission webhook. Mutating admission webhooks are invoked first, and can modify objects sent to the API server to enforce custom defaults. After all object modifications are complete, and after the incoming object is validated by the API server, validating admission webhooks are invoked and can reject requests to enforce custom policies.

For this series of the custom webhooks we going to write a webhooks to perform the following actions:

1. Check for the image tag of the pod (validation)
2. Check and mutate if the pod doesn't have any resource request and limits defined(mutation)
3. Inject environment variables to containers (mutation)

Here is the basic architecture of how the webhooks work:

![](https://cdn-images-1.medium.com/max/1600/1*kJWPA3feWUSeF_AhcR4-zQ.jpeg)

In nutshell, when you request for the new resource (pod in our case) to be created and if the pod has a predefined label, webhooks will pick the request and pass the request to the admission controller, and the admission controller directs the service to the external service for necessary changes, which will create a new patch. This patch is validated and then applied to the cluster to persist.

>Please the complete code for this series of tutorials at [https://github.com/Becram/kubernetes-webhook](https://github.com/Becram/kubernetes-webhook). I am a newbie to Golang and haven't written any tests.

I will not explain each part of the code but will try to explain the most important parts. Also, please note that we will use a lot of the modules and data structure from part 1 of the series.

First, we need to create a mutating webhook admission resource in our cluster, which is the resource in the cluster that defines how to pick the resource of mutation/validation and where to route the request for the modification.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: "kubernetes-webhook.acme.com"
webhooks:
  - name: "kubernetes-webhook.acme.com"
    objectSelector:
      matchLabels:
        mutation-check: enabled
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
        scope: "*"
    clientConfig:
      service:
        namespace: default
        name: kubernetes-webhook
        path: /mutate-pods
        port: 443
      caBundle: DEDACTED
    admissionReviewVersions: ["v1"]
    sideEffects: None
    timeoutSeconds: 2
```

Here, the clientConfig Part defines to route the request (kubernetes-webhook in this example)if the object has the label `mutation-check: enabled`.

>Note: Please note this request has to be a secured connection for which we need to create a `CA-signed certificate`, which can be generated with a script in the repo. skipped for brevity.

Of the three functions of this Golang-based service, lets me dive into the function of adding the container resource request and limit.

```go
// Mutate returns a new mutated pod according to lifespan tolerations rules
func (mpl containerResources) Mutate(pod *corev1.Pod) (*corev1.Pod, error) {
	mpl.Logger = mpl.Logger.WithField("mutation", mpl.Name())
	mpod := pod.DeepCopy()

	resources, err := parseResources()
	if err != nil {
		return &corev1.Pod{}, err
	}

	tn := corev1.ResourceRequirements{
		Limits:   resources.Limits,
		Requests: resources.Requests,
	}

	for index, n := range mpod.Spec.Containers {
		mpl.Logger.WithField("container", n.Name).
			Printf("applying default limits and request resource")

		mpod.Spec.Containers[index].Resources = tn

	}

	return mpod, nil
}
```

As discussed in part one we use corev1.Pod struct get the pod definition and modify/append the additional configuation to it. Here first we get the default resources defined and create a container resource definition variable `tn` (note is used `corev1.ResourceRequirements` struct). After we get the resource we append this resource by looping through all the containers in the container's array. On patched, the modified pod definition is returned and passed to the api server to apply to the cluster.

### **<span style="color:green"> References </span>**

1. https://slack.engineering/simple-kubernetes-webhook/
2. https://github.com/kubernetes/client-go
3. https://github.com/kubernetes/apimachinery
4. https://github.com/kubernetes/api
