---
layout: instance/blog-post
title: "Configuration management in the clouds"
tagline: "the return of cloud-init"
image: /assets/images/blog/cloud.jpg
---

When spinning up compute instances in the cloud, a proper and fully automated 
configuration management is a key success factor. I usually use [Ansible](https://github.com/ansible/ansible)
for the job as it's open source and widely adopted. 
But when it comes to highly volatile cloud infrastructure with instances 
spinning up and terminating on demand, configuration management
systems utilizing a control master (like Ansible, Puppet or [SaltStack](https://www.saltstack.com/))
can be cumbersome.

<!--more-->

## Hello cloud-init

This is where [cloud-init](https://cloudinit.readthedocs.io/) (or cloud-config) takes the
stage. I recently needed to provision AWS auto scaling groups made out of spot
instances.

This is perfectly doable using [Ansible Pull](https://docs.ansible.com/ansible/latest/cli/ansible-pull.html).
But Ansile needs a piece of software running upon system boot to initiate the 
Ansible Pull mechanism. I used _cloud-init_ for this and was (yet again) amazed by
it's simplicity. I had last used it years ago and went ahead provisioning
the whole system using cloud-init. I discovered some major advantages:

1. it _operates without a control master_,
1. it _runs upon system boot_ and is executed by the operating system,
1. is _applied once_ and the system is immutable afterwards and
1. is _supported by most_ operating systems and _cloud providers_.

The drawback is that it lacks an inventory. Thus inventory information has to be 
collected using the cloud providers' api. Nevertheless, most standard configuration
management tasks can be performed with cloud-init.

### Immutable infrastructure

[Terraform](https://www.terraform.io/docs/index.html) is currently the state-of-the-art solution
to set up infrastructure in a [multi-cloud workflow](https://www.lastweekinaws.com/podcast/screaming-in-the-cloud/episode-67-infrastructure-as-code-with-terraform-and-mitchell-hashimoto/). It focuses
on immutable infrastructure and will leave you with [cattle not pets](http://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/).

Fortunately Terraform has support for rendering Cloud-init manifests. The example below
supplies the _default.yaml_ cloud-init manifest as _user-data_ to a Digital Ocean Droplet.

```terraform
data "template_cloudinit_config" "this" {
  gzip = false
  base64_encode = false

  # users, groups, ssh keys
  part {
    content_type = "text/cloud-config"
    content      = file("manifests/default.yaml")
  }
}

# create droplet at digital ocean
resource "digitalocean_droplet" "this" {
  image    = data.digitalocean_image.centos.id
  name     = "sandbox"
  region   = "fra1"
  size     = "s-1vcpu-1gb"
  user_data = data.template_cloudinit_config.this.rendered
}
```

For reusable configuration management, you can 
[merge multiple cloud-init parts](https://cloudinit.readthedocs.io/en/latest/topics/merging.html#built-in-mergers).
In the example below, I first apply basic settings (user, ssh-keys), then
install an Nginx server and apply website configuration in the last part.
Those parts align pretty much with the Ansible role concept and are reusable
throughout other servers.

```terraform
# template cloud-init
data "template_cloudinit_config" "this" {
  gzip = false
  base64_encode = false

  # users, groups, ssh keys
  part {
    content_type = "text/cloud-config"
    content      = file("manifests/default.yaml")
  }

  # nginx web server
  part {
    content_type = "text/cloud-config"
    content      = file("manifests/nginx.yaml")
  }

  # website configuration
  part {
    content_type = "text/cloud-config"
    content      = templatefile("manifests/redirects.yaml", {
      redirects    = var.redirects
      certificates = acme_certificate.this
    })
  }
}
```

Each cloud-init part needs a header describing how to merge the different parts. By default,
sections of parts get replaced. The following _merge_how_ key merges part sections properly.

```yaml
#cloud-config
merge_how:
 - name: list
   settings: [append]
 - name: dict
   settings: [recurse_array]

users:
  - name: torsten
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    uid: 2005
    ssh_authorized_keys:
      - ssh-ed25519 AAAAC3NzaC1lZ..
```

You can find all cloud-init parts in my [personal infrastructure project](https://github.com/tboeghk/infrastructure/tree/master/redirect-railgun/manifests) as well as the [full Terraform setup](https://github.com/tboeghk/infrastructure/blob/master/redirect-railgun/main.tf).

## Summary

In my opinion cloud-inits' _simplicity focuses on provisioning the system_ with
just the right amount of information. Plus it does _not break the Terraform 
workflow_ as you do not need another tool for the job. 
[So maybe give cloud-init a chance!](https://cloudinit.readthedocs.io/en/latest/topics/examples.html)
