---
layout: post
title:  "Bare Metal install of CoreOS and Kubernetes with matchbox and terraform"
date:   2017-06-21 21:20:21 -0400
categories: jekyll update
---

So I decided to bare metal install a CoreOS cluster onto three physical Supermicro servers using the new matchbox/terraform packages.This is the announcement from the CoreOS guys (<https://coreos.com/blog/matchbox-with-terraform>) about it. I have used the bootcfg in the past from CoreOS with success and wanted to see how terraform would fit in the mix.

They have renamed bootcfg to matchbox in the new releases but it works pretty much the same way as bootcfg worked. In fact they renamed CoreOS to Container Linux as well. They must have hired some marketing guys. I decided to do the full kubernetes cluster which the CoreOS guys provided a great example called bootkube-install in the matchbox repo.

Your environment needs the following for all this to work:

* Matchbox v0.6+ installation with gRPC API enabled
* Matchbox provider credentials client.crt, client.key, and ca.crt
* PXE network boot environment
* Terraform v0.9+ and terraform-provider-matchbox installed locally on your system
* Machines with known DNS names and MAC addresses

You need to pull down Matchbox

```

$ git clone https://github.com/coreos/matchbox.git
$ cd matchbox/examples/terraform/bootkube-install

```

Once you are in the bootkube-install directory you need to change the terraform.tfvars file to suit your enviroment. Mine looks like this:

```

# This is the matchbox endpoint
matchbox_http_endpoint = "http://matchbox.mylab.local:8080"
matchbox_rpc_endpoint = "matchbox.mylab.local:8081"
ssh_authorized_key = "ssh-rsa **** PUT YOUR PUBLIC KEY HERE *******"

cluster_name = "mylabc5"
container_linux_version = "1353.7.0"
container_linux_channel = "stable"

# Machines
controller_names = ["srvc501"]
controller_macs = ["0c:c4:7a:3b:60:e4"]
controller_domains = ["srvc501.mylab.local"]
worker_names = ["srvc502", "srv503"]
worker_macs = ["0c:c4:7a:3b:6a:70", "0c:c4:7a:3b:61:04"]
worker_domains = ["srvc502.mylab.local", "srvc503.mylab.local"]

# Bootkube
k8s_domain_name = "mylabc5.mylab.local"
asset_dir = "assets"


install_disk = "/dev/sda"
# Optional
# container_linux_oem = ""
# experimental_self_hosted_etcd = "true"

```

After this is setup for your environment you need to run a terraform get to pull down the modules it needs.

```

terraform get

```

After that is done you can do check to see if everything will work by running

```

terraform plan

```

and finally terraform apply

```

terraform apply

```

The terraform apply command will generate all the files and put them in your matchbox directories in /var/lib/matchbox.

These are all the files terraform created:

```

root@util01:/var/lib/matchbox# ls -al *
assets:
total 2
drwxr-xr-x 3 matchbox matchbox 3 May 31 14:41 .
drwxr-xr-x 6 matchbox matchbox 6 May 31 15:30 ..
drwxr-xr-x 4 matchbox matchbox 4 Jun 20 17:10 coreos

groups:
total 11
drwxr-xr-x 2 matchbox matchbox   8 Jun 20 17:33 .
drwxr-xr-x 6 matchbox matchbox   6 May 31 15:30 ..
-rw-r--r-- 1 matchbox matchbox 834 Jun 20 17:12 cnfic5-cnfc5cnfi01.json
-rw-r--r-- 1 matchbox matchbox 775 Jun 20 17:12 cnfic5-cnfc5cnfi02.json
-rw-r--r-- 1 matchbox matchbox 775 Jun 20 17:12 cnfic5-cnfc5cnfi03.json
-rw-r--r-- 1 matchbox matchbox 830 Jun 20 17:32 container-linux-install-cnfc5cnfi01.json
-rw-r--r-- 1 matchbox matchbox 829 Jun 20 17:12 container-linux-install-cnfc5cnfi02.json
-rw-r--r-- 1 matchbox matchbox 829 Jun 20 17:12 container-linux-install-cnfc5cnfi03.json

ignition:
total 16
drwxr-xr-x 2 matchbox matchbox   10 Jun 21 15:51 .
drwxr-xr-x 6 matchbox matchbox    6 May 31 15:30 ..
-rw-r--r-- 1 matchbox matchbox 6625 Jun 20 17:12 bootkube-controller.yaml.tmpl
-rw-r--r-- 1 matchbox matchbox 4394 Jun 20 17:12 bootkube-worker.yaml.tmpl
-rw-r--r-- 1 matchbox matchbox  925 Jun 20 17:12 cached-container-linux-install.yaml.tmpl
-rw-r--r-- 1 matchbox matchbox  925 Jun 20 17:12 container-linux-install.yaml.tmpl
-rw-r--r-- 1 matchbox matchbox  810 May 31 18:21 coreos-install.yaml.tmpl
-rw-r--r-- 1 matchbox matchbox  669 Jun 20 17:12 etcd3-gateway.yaml.tmpl
-rw-r--r-- 1 matchbox matchbox  978 Jun 20 17:12 etcd3.yaml.tmpl
-rw-r--r-- 1 matchbox matchbox   99 May 31 15:30 simple.yaml.tmpl

profiles:
total 8
drwxr-xr-x 2 matchbox matchbox  10 Jun 21 15:51 .
drwxr-xr-x 6 matchbox matchbox   6 May 31 15:30 ..
-rw-r--r-- 1 matchbox matchbox  94 Jun 20 17:12 bootkube-controller.json
-rw-r--r-- 1 matchbox matchbox  86 Jun 20 17:12 bootkube-worker.json
-rw-r--r-- 1 matchbox matchbox 457 Jun 20 17:12 cached-container-linux-install.json
-rw-r--r-- 1 matchbox matchbox 501 Jun 20 17:12 container-linux-install.json
-rw-r--r-- 1 matchbox matchbox 473 May 31 18:21 coreos-install.json
-rw-r--r-- 1 matchbox matchbox  82 Jun 20 17:12 etcd3-gateway.json
-rw-r--r-- 1 matchbox matchbox  66 Jun 20 17:12 etcd3.json
-rw-r--r-- 1 matchbox matchbox  68 May 31 15:30 simple.json


```

Now to start the install you need to reboot the servers and set them to PXE boot.

```

ipmitool -H <serverIP> -U ADMIN -P ADMIN power off
ipmitool -H <serverIP> -U ADMIN -P ADMIN chassis bootdev pxe
ipmitool -H <serverIP> -U ADMIN -P ADMIN power on

```

Once booted the servers will pick up an address from the DHCP server and matchbox will direct it to install container Linux, docker and etcd.

The terraform program will still be looping waiting for container linux to install itself. Once installed it will attempt run install kubernetes.


Now this is the part of this that did not work for me after the install of container linux.
Terraform could not ssh to the machines to copy over all the kubernetes configuration files to complete the install. I could see this in the logs over and over again on the node servers.

```

Jun 21 16:06:19 srv503.mylab.local systemd[1]: Started OpenSSH per-connection server daemon (<utilIP>:55494).
Jun 21 16:06:19 srv503.mylab.local sshd[3340]: Connection closed by <serverIP> port 55494 [preauth]
Jun 21 16:06:22 srv503.mylab.local systemd[1]: Started OpenSSH per-connection server daemon (<utilIP>:55500).
Jun 21 16:06:22 srv503.mylab.local sshd[3345]: Connection closed by <serverIP> port 55500 [preauth]
Jun 21 16:06:25 srv503.mylab.local systemd[1]: Started OpenSSH per-connection server daemon (<utilIP>:55506).
Jun 21 16:06:25 srv503.mylab.local sshd[3350]: Connection closed by <serverIP> port 55506 [preauth]

```

hmm what is going on? Time for some debugging! A great way to debug terraform is to turn on logging. You can do that by passing a few environment variable to the shell.

```

export TF_LOG_PATH=./terraform.log
export TF_LOG=debug

```

When you run terraform apply again it dumps out piles and piles of logs....YA!! Logs, logs and more logs! so useful. So here is what it told me:

```

2017/06/21 20:08:33 [DEBUG] plugin: terraform: file-provisioner (internal) 2017/06/21 20:08:33 handshaking with SSH
2017/06/21 20:08:33 [DEBUG] plugin: terraform: file-provisioner (internal) 2017/06/21 20:08:33 handshaking with SSH
2017/06/21 20:08:34 [DEBUG] plugin: terraform: file-provisioner (internal) 2017/06/21 20:08:34 handshake error: ssh: handshake failed: ssh: unable to authenticate, attempted methods [none], no supported methods remain
2017/06/21 20:08:34 [DEBUG] plugin: terraform: file-provisioner (internal) 2017/06/21 20:08:34 Retryable error: ssh: handshake failed: ssh: unable to authenticate, attempted methods [none], no supported methods remain
2017/06/21 20:08:34 [DEBUG] plugin: terraform: file-provisioner (internal) 2017/06/21 20:08:34 handshake error: ssh: handshake failed: ssh: unable to authenticate, attempted methods [none], no supported methods remain
2017/06/21 20:08:34 [DEBUG] plugin: terraform: file-provisioner (internal) 2017/06/21 20:08:34 Retryable error: ssh: handshake failed: ssh: unable to authenticate, attempted methods [none], no supported methods remain

```

So obviously an SSH error and I found it odd it was not even trying the public key. Time to change that!

In matchbox/examples/terraform/bootkube-install/modules/bootkube/ is where all the terraform scripts live. The ssh.tf is the one we want to look at. Under the connection section (there are two of them) you need to pass your private key so ssh will try public key auth from your terraform server.

```

connection {
    type    = "ssh"
    host    = "${element(concat(var.controller_domains, var.worker_domains), count.index)}"
    private_key = "${file("/path/to/your/id_rsa")}" # add this line!!
    user    = "core"
    timeout = "60m"
  }

```

After that is done you can break the loop and do a terraform apply again it will actually authenticate and copy everything over for kubernetes.

You can check to see everything is working with kubectl when you move the kubeconfig file it creates to youe ~/.kube/config

```
root@util01:/var/lib/matchbox# kubectl get nodes
NAME                        STATUS    AGE       VERSION
srvc501.mylab.local   Ready     4h        v1.6.4+coreos.0
srvc502.mylab.local   Ready     4h        v1.6.4+coreos.0
srvc503.mylab.local   Ready     4h        v1.6.4+coreos.0

```

Of course checking a docker ps and a rkt list on the hosts will show you all the containers running kubernetes. Pretty great and the terraform bit is a welcome addition because it sure beats changing or modifing all the matchbox config files by hand.
