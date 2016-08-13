---
layout: post
title: Setting Up Chef Server on Ubuntu 14.04
---

Recently I was going through the process of setting up my own Chef Server on an Amazon EC2 server and ran into a bit of a snag resulting from how Ubuntu 14.04's internal hostname is setup:

```
...
================================================================================
Recipe Compile Error in /var/opt/opscode/local-mode-cache/cookbooks/private-chef/recipes/default.rb
================================================================================

Chef::Exceptions::ValidationFailed
----------------------------------
common_name is required
...
```

In this tutorial I'll walk you through setting up your own Chef Server on Ubuntu 14.04, with specific attention on setting up your hostname correctly and avoiding this error. 

As a first step, open AWS EC2, click "Launch Instance," and choose the Ubuntu 14.04 AMI. Next, make sure you choose an instance that has at least 4 GB of memory (e.g. `t2.medium`). Other than that, make sure you are launching your instance into a VPC that provides public DNS hostnames (this is the case for the default Amazon VPC and any VPC that you create with the VPC wizard) and that you use network settings that will allow you to access the server via SSH

When the instance is ready, go ahead and connect:

`$ ssh ubuntu@<your_instance_ip_address>`

Once logged in, switch to the `root` user:

`ubuntu@ip-xxx.xx.xx.xxx:~$ sudo su -`

At this point we need to install Chef Server. 

### Install Chef Server

1. Download the package
  - `$ wget https://packages.chef.io/stable/ubuntu/14.04/chef-server-core_12.8.0-1_amd64.deb`

2. Install with `dpkg`:
  - `$ dpkg -i chef-server-core_12.8.0-1_amd64.deb`

### Set Your Hostname

This part is the real reason for the tutorial. The installation instructions from Chef mention that our Chef Server host needs to have a Fully Qualified Domain Name (FQDN). Our Ubuntu instance isn't set up correctly for this by default and we can see that by running this command:

```
$ hostname -f
hostname: Name or service not known
```

Your EC2 instance has one if it was configured in the pattern laid out above, and it can be found in the EC2 console by selecting your instance and looking under the "Description" tab next to "Public DNS."

There are three places we need to use our public DNS address:

1. `/etc/hosts`
  - When you open this file you'll notice that it has `127.0.0.1 localhost`. Since `localhost` isn't a FQDN, we need to change this to our public DNS. When you're done, the top portion of your file should look something like this:
  
  `127.0.0.1 ec2-xx-xx-xxx-xxx.us-west-2.compute.amazonaws.com`

2. `/etc/hostname`
  - This file holds just the hostname, without reference to an IP address. When you open it on a new EC2 Ubuntu instance, it should just have somthing like this: `ip-xx-xx-xxx-xxx`. Remove that and add your public DNS address.

3. Finally, we need to run the `hostname` command to update our hostname without having to exit the system and log in again.
  - `$ hostname ec2-xx-xx-xxx-xxx.us-west-2.compute.amazonaws.com`

Now we should get a different result when we run the command above that gave us the "Name or service not known" error:

```
root@ip-xx-xx-xxx-xxx:~# hostname -f
ec2-xx-xx-xxx-xxx.us-west-2.compute.amazonaws.com
```

### Reconfigure Chef Server

Now that we've set up our hostname correctly, we can reconfigure the Chef Server for the first time without getting any errors:

```
root@ip-xx-xx-xxx-xxx:~# chef-server-ctl reconfigure
...
...
Chef Client finished, 393/464 resources updated in 01 minutes 57 seconds
Chef Server Reconfigured!
```

From here you can continue with the [official installation instructions](https://docs.chef.io/install_server.html), starting with step 5 in the **Standalone** section.

And that's it! Let me know in the comments if you have any questions or run into issues with these steps.
