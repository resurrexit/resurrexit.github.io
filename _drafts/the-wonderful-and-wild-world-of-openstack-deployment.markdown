---
layout: post
title: "The Wonderful and Wild World of OpenStack Deployment"
---

Disclaimer
----------

Before diving in, I'd like to state that these are my (an intern's) notes. There may
be inaccuracies or misunderstandings of various concepts or terms. If you notice
such, I would love it if you'd let me know. I get to learn from it, and you get
an attribution. Everyone wins!

First, some definitions
-----------------------

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

OpenStack deployment
--------------------

Currently, in terms of tools to make deployment easier, we have TripleO, which stands
for OpenStack on OpenStack. TripleO uses OpenStack as an undercloud (or seed) to deploy
the overcloud (this is what your users will be living). The undercloud is used to manage the
overcloud. See [this README] (http://docs.openstack.org/developer/tripleo-incubator/README.html)
for a great overview of the tech stack used in the project.




