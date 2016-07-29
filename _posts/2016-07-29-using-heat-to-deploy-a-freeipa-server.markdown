---
layout: post
title: "Using Heat to deploy a FreeIPA server"
date: 2016-07-29T15:12:10-04:00
---

Disclaimer/Entreaty
-------------------

If you see any technical errors or have any constructive criticism on how to improve the
template, _please_ contact me!

Preparation
------------

I've been working on a Heat template to provision a FreeIPA server. It is meant
to work alongside a [tripleo-quickstart](https://github.com/openstack/tripleo-quickstart)
deploy.

#### Change the default IP block the undercloud uses

In the previous post, I described debugging the problem caused by tripleo-quickstart's
choice of default IP block. Change it using
[this]({% post_url 2016-07-26-debugging-failed-undercloud-re-installs %}).

#### Ironic node creation

This solution does not create new Ironic nodes. So, if you already have deployed
your overcloud -- you'll need a free Ironic node. We could create a new one manually,
like what Lars Kellogg-Stedman describes
[here](http://blog.oddbit.com/2016/05/19/connecting-another-vm-to-your-tripleo-qu/).

#### Setting DNS server for the subnet

Depending on the network the VM is going to be attaching to, we may need to set the
DNS server for the subnet. Personally, I haven't found this to work (hence the hacky
solution of appending an `echo 'nameserver 8.8.8.8'` to `/etc/resolv.conf`). If anyone
has any suggestions on how to get DNS working in a better way, please send me an email!

I have tried this:

```
. ./stackrc  
neutron subnet-list  
neutron subnet-update <subnet_uuid> dns-nameserver=8.8.8.8  
```

but alas, it doesn't work as far as I can tell.

FreeIPA VM creation
-------------------

#### Heat stack creation

Now that we have an Ironic node to deploy our Heat template on, let's get started. Clone
[this repo](https://github.com/resurrexit/freeipa-heat-template) and tell Heat to create
a stack using the template:

```
git clone https://github.com/resurrexit/freeipa-heat-template
heat stack-create --template freeipa_server.yaml
```

#### Passing in parameters

All parameters in the template have defaults based on tripleo-quickstart.
Pass in parameters as needed with something like `-p DNSNameserver=8.8.8.8` _or_ by using
an environment file.[^1]

After cloud-init completes, you should have a working FreeIPA server!

Footnotes
---------

[^1]: [An Introduction to OpenStack Heat](http://blog.oddbit.com/2013/12/06/an-introduction-to-openstack-heat/)
