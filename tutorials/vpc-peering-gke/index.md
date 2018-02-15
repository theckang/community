---
title: Set up VPC Network Peering with Google Kubernetes Engine
description: Learn how to connect Google Kubernetes Engine clusters in separate VPC networks using VPC network peering.
author: theckang
tags: Google Kubernetes Engine, VPC Network Peering
date_published:
---
This tutorial shows how to connect Google Kubernetes Engine clusters in separate VPC networks using VPC network peering.

There are scenarios in which you may have clusters in separate VPC networks.  These networks can be in the same or different projects.  If in different projects, the projects can be associated with no organization, one organization, or different organizations.

By default these clusters cannot communicate because they are in separate VPC networks.  However, you may want to expose services to a cluster in another network, project, or organization.  The options to do so include:

* Internet
* Cloud VPN
* VPC Network Peering

VPC Network Peering allows you to make services available across VPC networks in the private RFC1918 space without having to expose services to the public Internet or create and manage VPNs.

In this tutorial, you create clusters in two VPC networks in one project, peer the networks, and expose a Kubernetes service using an Internal Load Balancer.  The following diagram shows the architecture:

## Objectives

* Create VPC networks and subnets.
* Create Kubernetes Engine clusters.
* Peer the VPC networks.
* Configure firewall rules.
* Deploy Kubernetes internal services to test access to services.

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

The Internal Load Balancer is a regional product so the subnets for each cluster need to be in the same region.  The subnet IPv4 address range cannot overlap for peering to work.

1. Create the VPC networks:

        gcloud compute networks create network-a --subnet-mode=custom
        gcloud compute networks create network-b --subnet-mode=custom

1. Create the subnets:

        gcloud compute networks subnets create subnet-a --network=network-a --range=192.168.0.0/16 --region=us-west1
        gcloud compute networks subnets create subnet-b --network=network-b --range=10.0.0.0/16 --region=us-west1


## Create Kubernetes Engine clusters

1. Create the Kubernetes Engine cluster in network-a



1. Create the Kubernetes Engine cluster in network-b



## Cleaning up

