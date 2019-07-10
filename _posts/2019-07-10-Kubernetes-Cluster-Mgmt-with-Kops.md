---
title:  "Kubernetes Cluster Management with kops"
last_modified_at: 2019-04-15T16:00:58-04:00
categories:
  - DevOps
tags:
  - Kubernetes
  - Orcheastration
header:
  teaser: /assets/images/kops.png

toc: true
toc_label: "What is here"
---



# Kubernetes Cluster Management with kops
You may find the complete script [here](https://gist.github.com/Becram/d68009a56fff139bbfad982ffb5c204f)

After Google donated its amazing container orcheastration tool, kubernetes to Cloud Native Computing foundation (CNCF), many devops breath the air of sigh. Kubernetes with it amazing power took over the devops world which made the devops more powerful, resilient and highly scalable with great implimentation of fault tolerance.

All these power come at a price. For a newbies kubernetes cluster creation seems a additional headache with very steep learning curves. Manual cluster creation seems a great challenge and tedious task. So, this is where the open source [kops](https://github.com/kubernetes/kops) comes to g rescue. With plethora of task abstraction, kops is a great tool to create kubernetes cluster. Kops prvides the production ready cluster with high scalability on the popular cloud architecture as AWS and Google cloud. Moreover it has a close ressemblence with *kubectl* syntax which makes learning very flat.


## Key features kops

* Automates the provisioning of Kubernetes clusters in AWS and GCE
* Deploys Highly Available (HA) Kubernetes Masters
* Built on a state-sync model for **dry-runs** and automatic **idempotency**
* Ability to generate Terraform
* Supports custom Kubernetes add-ons
* Command line autocompletion
* YAML Manifest Based API Configuration
* Templating and dry-run modes for creating
 Manifests
* Choose from eight different CNI Networking providers out-of-the-box
* Supports upgrading from kube-up
* Capability to add containers, as hooks, and files to nodes via a cluster manifest


### Without further ado lets dive into kops (For this tutorial we will be using AWS cloud)

First thing first make sure you have aws account. So, lets install aws cli as shown below:

```
    sudo apt-get update
    # install python-pip
    sudo apt install python-pip
    #install AWS client
    pip install awscli --upgrade --user
    #install jq
    sudo apt-get install jq
```
After that make sure you have configured *aws cli* with appropriate configuration using command *aws configure*

Next we are goiing to install kops. kops installation is very easy

```
    # Installing kops version 1.12.2
    wget https://github.com/kubernetes/kops/releases/download/1.12.2/kops-linux-amd64
    chmod +x kops-linux-amd64
    sudo mv kops-linux-amd64 /usr/local/bin/kops
```
 We need subdomain name for the cluster which will be the identity of the cluster. Since we dont have  registered subdomain we can have as domain name as *becram.k8s.local* for our case. All the configuration needs to be stored in the cloud itself for which we use AWS S3 bucket. lets create a S3 buket as:

```
    SUBDOMAIN_NAME=becram.k8s.local
    AWS_DEFAULT_REGION=us-east-2 # I use us-east-2 region
    aws s3 rb s3://$SUBDOMAIN_NAME-kubernetes-state
    aws s3 mb s3://$SUBDOMAIN_NAME-kubernetes-state --region $AWS_DEFAULT_REGION
    export KOPS_STATE_STORE=s3://$SUBDOMAIN_NAME-kubernetes-state
```
Next we need to have a ssh keys to access the server nodes. We create  keys as:

```
    KOPS_HOME=/home/bikram/kops
    SSH_KEY_HOME=$KOPS_HOME/$SUBDOMAIN_NAME/sshkeys
    # create ssh key home
    mkdir $SSH_KEY_HOME
    # generate ssh key
    ssh-keygen -f $SSH_KEY_HOME/id_rsa -t rsa #save the key in the sshkeys
```

Finally lets the party begin. lets run the cluster as
```
    SSH_PUBLIC_KEY=$SSH_KEY_HOME/id_rsa.pub
    kops create cluster --v=0 \
        --cloud=aws \
        --node-count 2 \
        --master-size=t2.medium \
        --master-zones=us-east-2a \
        --zones us-east-2a,us-east-2b \
        --name=$SUBDOMAIN_NAME \
        --node-size=t2.micro \
        --ssh-public-key=$SSH_PUBLIC_KEY \
        2>&1 | tee $KOPS_HOME/create_cluster.txt

    kops update cluster $SUBDOMAIN_NAME --yes
    ############# UPDATE CLUSTER ENDS ################"

```
Your cluster is ready.

