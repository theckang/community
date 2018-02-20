---
title: Set up VPC Network Peering with Google Kubernetes Engine
description: Learn how to connect Google Kubernetes Engine clusters in separate VPC networks using VPC network peering.
author: theckang
tags: Google Kubernetes Engine, VPC Network Peering
date_published:
---
This tutorial shows how to connect Google Kubernetes Engine clusters in separate VPC networks using VPC network peering.

There are scenarios in which you may have clusters in separate VPC networks.  These networks can be in the same or different projects.  If the networks are in different projects, the projects can be associated with no organization, one organization, or different organizations.

By default these clusters cannot communicate because they are in separate VPC networks.  However, you may want to expose services to a cluster in another network, project, or organization.  The options to do so include:

* Internet
* Cloud VPN
* VPC Network Peering

VPC Network Peering allows you to make services available across VPC networks in the private RFC1918 space without having to expose services to the public Internet or create and manage VPNs.

In this tutorial, you create clusters in two VPC networks in one project, peer the networks, and expose a Kubernetes service using an Internal Load Balancer.  The following diagram shows the architecture:



The peering steps are similar if the VPC networks are in separate projects or organizations.  This is highlighted in the tutorial.

## Objectives

* Create VPC networks and subnets.
* Create Kubernetes Engine clusters.
* Peer the VPC networks.
* Configure firewall rules.
* Deploy Kubernetes internal services.
* Test access to internal services.

## Costs

This tutorial uses billable components of Google Cloud Platform, including:

* Google Kubernetes Engine
* Google Cloud Load Balancing

Use the [Pricing Calculator][pricing] to generate a cost estimate based on your
projected usage.

[pricing]: https://cloud.google.com/products/calculator

## Before you begin

1. Create a [Google Cloud Platform project](https://console.cloud.google.com/project).

1. Enable [billing](https://console.cloud.google.com/billing) for the project.

1. Enable the [Google Kubernetes Engine API](https://console.cloud.google.com/flows/enableapi?apiid=container.googleapis.com).

1. Open [Cloud Shell](https://console.cloud.google.com/cloudshell).

## Create VPC networks and subnets

The Internal Load Balancer is a regional product so the subnets for each cluster need to be in the same region.  The subnet IPv4 address ranges cannot overlap for peering to succeed.

1. Create the VPC networks:

        gcloud compute networks create network-a --subnet-mode=custom
        gcloud compute networks create network-b --subnet-mode=custom

1. Create the subnets:

        gcloud compute networks subnets create subnet-a --network=network-a --range=192.168.0.0/16 --region=us-west1
        gcloud compute networks subnets create subnet-b --network=network-b --range=10.0.0.0/16 --region=us-west1

## Create Kubernetes Engine clusters

Although the networks are in the same region, the clusters can be created in different zones.

1. Create the Kubernetes Engine clusters in each VPC in separate zones:

        gcloud container clusters create cluster-a --network network-a --subnetwork subnet-a --zone us-west1-a
        gcloud container clusters create cluster-b --network network-b --subnetwork subnet-b --zone us-west1-b

## Peer the VPC networks

Each network must establish a peering association.

1. Peer network-a with network-b:

        gcloud compute networks peerings create peer-ab --network=network-a --peer-network=network-b --auto-create-routes

    You should see in the output `state: INACTIVE` as the peering association from `network-b` is not created.

    If the network is in another project or organization, you would execute:

        gcloud compute networks peerings create peer-ab --network=network-a --peer-network=network-b --peer-project=[NETWORK-B-PROJECT-NAME]

1. Peer network-b with network-a:

        gcloud compute networks peerings create peer-ba --network=network-b --peer-network=network-a --auto-create-routes

    You should see in the output `state: ACTIVE` as the peering associations on both networks are established.

    If the network is in another project or organization, you would execute:

        gcloud compute networks peerings create peer-ba --network=network-b --peer-network=network-a --peer-project=[NETWORK-A-PROJECT-NAME]

## Configure the firewall rules

Google Kubernetes Engine automatically creates default firewall rules when you create the cluster.  The `ssh` rule allows communication between the master and workers.  The `all` rule allows communication between pods in the cluster.  The `vms` rule allows communication between the nodes in the cluster.

1. View the current firewall rules:

        gcloud compute firewall-rules list --filter 'network-a OR network-b'

   You should see three firewall rules (all, ssh, vms) per cluster in the output:

        NAME                        NETWORK    DIRECTION  PRIORITY  ALLOW                         DENY
        gke-cluster-a-xxxxxxxx-all  network-a  INGRESS    1000      tcp,udp,icmp,esp,ah,sctp
        gke-cluster-a-xxxxxxxx-ssh  network-a  INGRESS    1000      tcp:22
        gke-cluster-a-xxxxxxxx-vms  network-a  INGRESS    1000      icmp,tcp:1-65535,udp:1-65535
        gke-cluster-b-xxxxxxxx-all  network-b  INGRESS    1000      ah,sctp,tcp,udp,icmp,esp
        gke-cluster-b-xxxxxxxx-ssh  network-b  INGRESS    1000      tcp:22
        gke-cluster-b-xxxxxxxx-vms  network-b  INGRESS    1000      icmp,tcp:1-65535,udp:1-65535

1. Create a firewall rule to allow `network-a` to access internal services over port 80 on `network-b`:

        gcloud compute firewall-rules create fw-ab --allow=TCP:80 --source-ranges 192.168.0.0/16  --network network-b

## Deploy Kubernetes internal services

Deploy a hello application service to `cluster-b`.

1. Set the zone of `cluster-b`:

        gcloud config set compute/zone us-west1-b

1. Get authentication credentials to `cluster-b`:

        gcloud container clusters get-credentials cluster-b

1. Create the deployment:

        kubectl run hello-server --image gcr.io/google-samples/hello-app:1.0 --port 8080

1. Expose the service using the Internal Load Balancer:

        cat <<EOF | kubectl apply -f -
        apiVersion: v1
        kind: Service
        metadata:
          name: hello-server-svc
          annotations:
            cloud.google.com/load-balancer-type: "Internal"
          labels:
            run: hello-server
        spec:
          type: LoadBalancer
          ports:
          - port: 80
            targetPort: 8080
            protocol: TCP
          selector:
            run: hello-server
        EOF

1. Watch the service until the `EXTERNAL-IP` is provisioned.

        kubectl get svc hello-server-svc --watch

1. You should see in the output:

        NAME               TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
        hello-server-svc   LoadBalancer   xx.x.xxx.xxx   x.x.x.x      80:xxxxx/TCP   2m

1. Exit the watch with `CTRL+C`.  Save the `EXTERNAL-IP` in a shell variable:

        export HELLO_SVC=$(kubectl get svc hello-server-svc -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

## Test access to internal services

Test access to the hello application in `cluster-b` from `cluster-a`.

1. Set the zone of `cluster-a`:

        gcloud config set compute/zone us-west1-a

1. Get authentication credentials to `cluster-a`:

        gcloud container clusters get-credentials cluster-a

1. Create container and execute a shell session:

        kubectl run -i --rm --tty busybox --image=radial/busyboxplus:curl --env="HELLO_SVC=$HELLO_SVC" --restart=Never -- sh

1. From the container in `cluster-a`, test access to the internal service in `cluster-b`:
        # curl HELLO_SVC



## Cleaning up




