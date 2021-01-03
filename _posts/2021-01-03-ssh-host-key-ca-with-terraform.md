---
layout: instance/blog-post
title: "SSH host key CA with Terraform"
tagline: "Securing your VPC with a SSH host key CA"
image: /assets/images/blog/zion.jpg
featured: true
---

When dealing with volatile infrastructure, we treat servers
as cattle not pets. In a VPC, jumphost servers might even 
be launched on demand only. To ensure that we only connect
to trusted servers launched by us, we can establish a 
Certificate Authority for our SSH host keys. 

<!--more-->

## SSH host keys

Each SSH server instance identifies itself with a unique SSH
host key. Upon every login, this host key is presented to the
user:

`````
$ ssh in-awe.fra1.o11ystack.org
The authenticity of host 'in-awe.fra1.o11ystack.org (157.245.24.111)' can't be established.
ECDSA key fingerprint is SHA256:EinMrd6XfGaRBgP2hnu...
Are you sure you want to continue connecting (yes/no/[fingerprint])? 
`````

In reality, nobody really checks the SSH host key upon 
login. The message is either always discarded with _yes_
or configured away in your _~/.ssh/config_ via

````ssh
StrictHostKeyChecking no
````

The following message should lead to a heads up, as the host
key presented does not match the host key in you _/.ssh/known_hosts_
file.

````
$ ssh in-ewe.fra1.o11ystack.org
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@       WARNING: POSSIBLE DNS SPOOFING DETECTED!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
The ECDSA host key for in-ewe.fra1.o11ystack.org has changed,
and the key for the corresponding IP address 157.245.24.111
has a different value. This could either mean that
DNS SPOOFING is happening or the IP address for the host
and its host key have changed at the same time.
Offending key for IP in /Users/torsten/.ssh/known_hosts:114
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ECDSA key sent by the remote host is
SHA256:EinMrd6XfGaRBgP2hnu0z...
Please contact your system administrator.
[...]
````

Again, reality strikes again: The mismatching lines gets deleted
from the _/.ssh/known_hosts_ file and voila: everything is fine. 
Or is it?

## SSH host key CA

The basic idea is to have a Certificate Authority (CA) signing 
each SSH host key. The CA's public key is known to the client
and can be used to verify the authenticity of the server.

### Creating CA keys

Log in to any server (or use your client machine).
Create a _ECDSA_ key pair and store the private and public key.

````bash
$ ssh-keygen -f /etc/ssh/ca -t ecdsa
$ cat /etc/ssh/ca
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktd...
-----END OPENSSH PRIVATE KEY-----
$ cat /etc/ssh/ca.pub
ecdsa-sha2-nistp256 AAAAE2VjZHNh...
````

### Signing host keys

As we are dealing with volatile infrastructure, we want to sign every
SSH host key of every server we create. I use 
[cloud-init to set up servers](https://www.thiswayup.de/blog/2020/configuration-management-in-the-clouds.html)
as it integrates seamlessly with [Terraform](https://registry.terraform.io/providers/hashicorp/template/latest/docs/data-sources/cloudinit_config) and ~~most~~ all cloud providers.

````yaml
#cloud-config
merge_how:
 - name: list
   settings: [append]
 - name: dict
   settings: [recurse_array]

write_files:
  - path: /etc/ssh/ca
    owner: root:root
    permissions: '0600'
    content: |
      -----BEGIN OPENSSH PRIVATE KEY-----
      b3BlbnNzaC1rZXktd...
      -----END OPENSSH PRIVATE KEY-----
  - path: /etc/ssh/ca.pub
    owner: root:root
    permissions: '0644'
    content: ecdsa-sha2-nistp256 AAAAE2VjZHNh...

runcmd:
  # sign and configure ssh host keys
  - ssh-keygen -s /etc/ssh/ca -I "$(hostname)" -n "$(hostname),$(hostname -I|tr ' ' ',')$(hostname).<FQDN>" -V -5m:+520w -h /etc/ssh/ssh_host_rsa_key.pub /etc/ssh/ssh_host_ecdsa_key.pub /etc/ssh/ssh_host_ed25519_key.pub

  # configure ca usage on the server
  - echo "@cert-authority * $(cat /etc/ssh/ca.pub)" >> /etc/ssh/ssh_known_hosts

  # configure new signed host certificates
  - for i in /etc/ssh/ssh_host*_key-cert.pub; do echo "HostCertificate $i" >>/etc/ssh/sshd_config; done

  # remove ssh key signing private key
  - rm -f /etc/ssh/ca

  # restart sshd to make changes have effect
  - systemctl restart sshd
````

### Hostname principals

When signing the key you need to configure the principals (_-n_)
(FQDNs and server names) the host can be reached with. The example
above registers the hostname and the IPs configured. 

````bash
$ hostname
in-awe
$ hostname -I|tr ' ' ','
157.230.28.200,10.17.0.4,10.135.0.1,2a03:b0c0:...
````

You might want to add the host's FQDNs (_hostname -f_) to the list.

### Certifcate validity

The validity is given as a validity interval (for server time skew).
In the example above the certificate is valid for 5 years.

````
-V -5m:+520w
````

## Caveats

Well, we are copying the CA's private key to the server. I'd love to
avoid that and precompute the host's SSH keys, but unfortunately this
is currently not possible in plain Terraform (only using external 
data providers).