---
layout: instance/blog-post
title: "Digital Ocean as cloud provider"
tagline: "project report from a cloud migration to Digital Ocean"
#image: /assets/images/blog/cloud.jpg
featured: true
---

I recently migrated a customer from it's own data center to
Digital Ocean's FRA1 data center. As there a little to no
marketing free hands on migration use cases out there, here's
finally one ;-)

<!--more-->

## Tools & techniques used

To get a glimpse of the size of the setup, it utilizes roughly _30+ instances_
mostly with _shared cpus_ (basic plan). All ingress traffic is routed through
Digital Ocean loadbalancers (small) into a VPC. Except for a bastion host, all
ingress traffic is denied on the host's public interface.

The whole setup is fully automated using [Terraform as cloud provisioner](https://registry.terraform.io/providers/digitalocean/digitalocean/latest/docs) 
and [Cloud-Init as server provisioner](https://www.thiswayup.de/blog/2020/configuration-management-in-the-clouds.html). 

## Drivers for DO

The customer decided to go with Digital Ocean due to their simplicity and
their attractive pricing model. 

## Caveats

### Shared vs. dedicated CPU

- price range for shared cpu very attractive
- 10% more power for dedicated cpu

### Loadbalancers

- public only
- no flaoting ip assignable

### DNS

- data center subdomain delegated to digital ocean
- very slow, almost unusable
- delay up to 15 minutes
- deploy unbound cache in your vpc
