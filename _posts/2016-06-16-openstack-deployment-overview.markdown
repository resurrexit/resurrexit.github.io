---
layout: post
title: "From my notes: OpenStack deployment overview"
date: 2016-06-16T16:02:54-04:00
---

Disclaimer
----------

Before diving in, I'd like to state that these are my (an intern's) notes. There may
be inaccuracies or misunderstandings of various concepts or terms. If you notice
such, I would love it if you'd let me know. I get to learn from it, and you get
an attribution. Everyone wins!


OpenStack deployment
--------------------

Currently, in terms of tools to make deployment easier, we have TripleO, which stands
for OpenStack on OpenStack. TripleO uses OpenStack as an undercloud (or seed) to deploy
the overcloud (this is where your users will be living). The overcloud is where the
guest virtual machines will be running, and where all the OpenStack services you need for
your users will be running. Simply put, the overcloud is the functional cloud, and the
undercloud is used to get you there.

The undercloud is used to deploy and manage the overcloud. The undercloud is a
single-node OpenStack installation on bare metal. It contains the OpenStack services
essential to deploying the overcloud. These services are Nova, Neutron,
Glance, Keystone, Ironic, Heat, Horizon, and Ceilometer. Each one of the services listed here
provide something essential to deploy the overcloud; for example, Glance handles the disk
images, while Neutron is in charge of network connectivity. For all OpenStack services, we
have Keystone for authentication and authorization, and Horizon is a modular web-based interface
built with Django. Ironic is for OpenStack bare metal provisioning, and so on.

{% img left /assets/images/marketecture-diagram.png 'openstack diagram' %}
[Image source](http://docs.openstack.org/security-guide/introduction/introduction-to-openstack.html)

See [README](http://docs.openstack.org/developer/tripleo-incubator/README.html)
for a great overview of the tech stack used in the project. In short, the installation is managed
Heat, a template-based orchestration engine. It acts as the puppet-master in the post deployment
phase. Heat calls Puppet, passing data for Hiera to find the appropriate module. Remember that
Puppet is designed for handling configuration management for a large number of systems. Hiera is a
tool for key/value lookup, enabling us to separate code from data.

Before you run TripleO, however, you need to have an environment set up for it.
[Tripleo-quickstart](https://github.com/openstack/tripleo-quickstart) is a set of Ansible playbooks
that calls the necessary setup code before calling TripleO.

Some definitions
----------------

PXE booting -- remember the boot sequence menu that you can trigger upon booting
a computer by pressing F12? Okay, so it may not be F12 on your particular machine,
but there's *some* key or key combination to get to that menu. Generally, there
will be some options to boot off of USB, hard disk, and the network. When you
select the network option, the computer will search for a PXE server to pull a
software image off of. The good thing about PXE is that it relies on two
industry-standard network protocols, namely DHCP and TFTP (trivial FTP).

Hypervisor -- computer software, firmware, hardware *dedicated* to running virtual
machines. Note: OpenStack is *not* a hypervisor. See
[this](http://blog.rackspace.com/three-openstack-myths-to-consider-as-we-close-out-2013/)
for an explanation why. If you are too lazy to visit that link, OpenStack does not
replace your current platforms. Instead, Nova is designed to talk to multiple kinds
of hypervisors. It works *alongside* your usual hypervisors, like KVM, ESXi, and so on.

Provisioning -- getting an image onto the system. Hence, the term bare metal provisioning
refers to the process of getting an image onto bare metal, also known as the computer's
hardware.

Further Reading
---------------
[OpenStack Folsom Architecture](http://ken.pepple.info/openstack/2012/09/25/openstack-folsom-architecture/)[^1]

Footnotes
---------
[^1]: Folsom is already EOL, but the descriptions of the services are mostly good. The network service has been renamed to Neutron.
