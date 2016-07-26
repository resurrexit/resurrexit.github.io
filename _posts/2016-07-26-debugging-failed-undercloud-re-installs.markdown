---
layout: post
title: "Debugging failed undercloud (re)installs"
date: 2016-07-26T12:01:49-04:00
---

Introduction
------------

I am in the process of writing a Heat template that creates and provisions a FreeIPA server.
The Heat template uses an OS::Heat::SoftwareConfig encapsulating a bash script to prepare for,
install, and configure FreeIPA. When the VM reaches the step of running `ipa-server-install`, it
fails with:

```
2016-07-20T14:23:12Z WARNING Invalid IP address 192.0.2.19 for identity.warp.lab4.eng.bos.redhat.com: cannot use IANA reserved IP address
```

It turns out that this is not a problem with my script, but rather a problem with the IP range that
tripleo-quickstart uses by default for the undercloud. The block `192.0.2.0/24` has been
assigned for use in documentation and example code only. See [here](https://tools.ietf.org/html/rfc5735)
for more details.

FreeIPA doesn't accept IP addresses in that range. As such, we need to re-install our undercloud with
a different block. In fact, Juan Osorio has been trying to get this [patch](https://review.openstack.org/#/c/343443/) in.

Problem
-------

First, we'll generate a new `undercloud.conf` using [this](http://ucw-bnemec.rhcloud.com/). It's not
quite as simple as editing `network_cidr` in the configuration file, as all the network settings need
to match up. Thus, the tool I'm linking to is a helpful way to ensure that they are valid.

Because I was unaware that one had to clean up the neutron resources before re-installing, I ran
`openstack undercloud install` after I put my new `undercloud.conf` in the home directory. The install
fails.

Debugging
---------

#### Step 1: Information gathering

The error log for the install can be found in `$HOME/.instack/install-undercloud.log`. Let's scroll down. We see error messages that look like
this:

```
New cidr 192.168.0.0/24 does not equal old cidr 192.0.2.0/24
INFO: + echo 'Will attempt to delete and recreate subnet 45e7fc6b-aa7c-4cf7-b191-fd475cf81e02'
INFO: Will attempt to delete and recreate subnet 45e7fc6b-aa7c-4cf7-b191-fd475cf81e02
```

and subsequently

```
INFO: Unable to complete operation on subnet 45e7fc6b-aa7c-4cf7-b191-fd475cf81e02: One or more ports have an IP allocation from this subnet.
INFO: Neutron server returns request_ids: ['req-56b86273-cf70-4a8b-9506-8be6ea03fe38']
INFO: (os-refresh-config) [ERROR] during post-configure phase. [Command '['dib-run-parts', '/usr/libexec/os-refresh-config/post-configure.d']' returned non-zero exit status 1]
INFO: (os-refresh-config) [ERROR] Aborting...
```
[^1]

#### Step 2: Forming a hypothesis

As we can see, the install aborted in the post-configure phase because we still have ports that have an IP allocation from the subnet that
needs to be cleaned up. If we can do the clean-up now, then we can move on to re-running the undercloud install.

Commands like `nova list` and `neutron net-list` aren't responding. This is a problem, since we'll need Neutron to respond. Is it the same
problem we encountered before? If it is, then we know that this is because Keystone is down or not responding properly. Let's try running a
command to the identity service, like `openstack token issue`. We get this error message:

```
Discovering versions from the identity service failed when creating the password plugin. Attempting to determine version from URL.
Unable to establish connection to http://192.0.2.1:5000/v2.0/tokens
```

#### Step 3: Verifying the truth value of this hypothesis

Before we go any further into the rabbit hole, let's restart relevant services to ensure that this isn't the same error as before. 

```
sudo systemctl restart rabbitmq-server.service
sudo systemctl restart httpd.service
```

It's still not working. Time to take a look at `stackrc` to see if we can garner any clues.

```
export OS_AUTH_URL=http://192.0.2.1:5000/v2.0/
```

We should check to see if and/or where Keystone is listening.

```
netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 192.168.0.1:5000        0.0.0.0:*               LISTEN     
```

Keystone service is no longer listening at 192.0.2.1. It is listening on 192.168.0.1:5000. To test this, we can poke it to see if it responds:

```
[stack@undercloud ~]$ curl 192.168.0.1:5000
{"versions": {"values": [{"status": "stable", "updated": "2016-04-04T00:00:00Z", "media-types": [{"base": "application/json", "type": "application/vnd.openstack.identity-v3+json"}], "id": "v3.6", "links": [{"href": "http://192.168.0.1:5000/v3/", "rel": "self"}]}, {"status": "stable", "updated": "2014-04-17T00:00:00Z", "media-types": [{"base": "application/json", "type": "application/vnd.openstack.identity-v2.0+json"}], "id": "v2.0", "links": [{"href": "http://192.168.0.1:5000/v2.0/", "rel": "self"}, {"href": "http://docs.openstack.org/", "type": "text/html", "rel": "describedby"}]}]}}
```

#### Step 4: Fixing the [current] problem

We have a response from Keystone! It is alive and well, but we haven't been using the right address. Edit the `OS_AUTH_URL` variable
in stackrc to match the new address. Now, calling `openstack token issue` works.

Continuing on
-------------

The undercloud install is still in a failed state. All we've done so far is fix the services that broke as a result of the
install failing halfway. We still need to clean up and restart the the undercloud install. According to 
[this bugzilla](https://bugzilla.redhat.com/show_bug.cgi?id=1228862)[^2], we have to delete the Neutron resources by hand.

First, delete all the ports here:

```
neutron port-list
neutron port-delete [port_id]
```

Delete the subnet.

```
neutron subnet-list
neutron subnet-delete [subnet_id]
```

Delete the network itself.

```
neutron net-list
neutron net-delete ctlplane
```

Now, we can re-run `openstack undercloud install`. Don't worry about the fact that we deleted ctlplane; the undercloud install will recreate it for
us.

For further reference
---------------------

Quoting the tripleo installation docs[^3], "openstack undercloud install can be rerun to reapply changes from undercloud.conf to the undercloud. 
Note that this should not be done if an overcloud has already been deployed or is in progress." This is what I did wrong. While I did not have a
full-blown overcloud deployed, I did have a node (the FreeIPA one I've been working on) sitting on the overcloud, on ctlplane. I _think_ this is
what caused the error. Next time, when rerunning the undercloud install, remember to clean up any resources that may cause issues.

To be continued...
------------------
After running the undercloud install (which succeeded, by the way), we came across some strangeness in ironic. There will be an upcoming post on
the issues we've seen.


Footnotes
---------
[^1]: Date and time stamps redacted for brevity.
[^2]: Specifically, comment #9 is helpful to us. The rest of the discussion is still very helpful, however.
[^3]: [Tripleo installation docs](http://docs.openstack.org/developer/tripleo-docs/installation/installation.html)

