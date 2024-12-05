---
authors:
- Hugo Authors
date: "2024-09-01"
excerpt: Creating static public IP on Google Cloud Platform
hero: /images/on_gcp_staticip/cover-img.png
title: On Creating Static Public IP on GCP
---

Sometimes when we are working with Public Cloud, we need to have a static public IP address for our instances. This is useful when we want to whitelist the IP address on some services or when we want to access the instances from the internet. In this post, we will learn how to create a static public IP address on Google Cloud Platform.
To have a static public IP address on Google Cloud Platform, we will first create a VPC network and then create Cloud NAT to allow the instances to access the internet.


The workflow is as below - 
<p align="center">
<img alt = 'png' src='/images/on_gcp_staticip/workflow.png'/>
</p>

<u><b>
    <p style="font-size:20pt ">
      Setting Up VPC 
    </p>
</b></u>

On the GCP console, navigate to the `VPC network` > `Create VPC network`.

<p align="center">
<img alt = 'png' src='/images/on_gcp_staticip/create-vpc.png'/>
</p>

<p align="center">
<img alt = 'png' src='/images/on_gcp_staticip/create-vpc2.png'/>
</p>

<u><b>
    <p style="font-size:20pt ">
      Setting Up Firewall Rules
    </p>
</b></u>

In case we will want to allow SSH for Bastion Host from specific IPs, we will need to create a firewall rule to allow SSH traffic. To do this, navigate to the `VPC network` > `Firewall rules` > `Create Firewall Rule`.

We will be using the below config

```md
Direction of traffic: Ingress
Action on match: Allow
Targets tags: http
Source IP ranges: 0.0.0.0/0
Protocols and ports: tcp:80
```

<p align="center">
<img alt = 'png' src='/images/on_gcp_staticip/create-firewallrule.png'/>
</p>

<p align="center">
<img alt = 'png' src='/images/on_gcp_staticip/create-firewallrule2.png'/>
</p>

<u><b>
    <p style="font-size:20pt ">
      Setting Up Cloud NAT
    </p>
</b></u>

Navigate to the `Network Services` > `Cloud NAT` > `Create Cloud NAT gateway`.

We will be using the below config

```md
NAT type: Public
VPC network: <VPC network we created above>
Region: <Same Region as VPC>
Cloud Router: Create a Cloud Router
Cloud NAT Mapping: Source endpoint type: VM instances, GKE nodes, Serverless
Source subnets & IP ranges: All subnets' primary and secondary IP ranges
NAT IP address: Manual > Reserve a new Static IP address
```

<p align="center">
<img alt = 'png' src='/images/on_gcp_staticip/create-nat.png'/>
</p>

<p align="center">
<img alt = 'png' src='/images/on_gcp_staticip/create-nat2.png'/>
</p>

The Cloud NAT IP is the static public IP address that we will use to whitelist on services or access the instances from the internet.


That’s it for now.

Thank you for reading and have a nice day!