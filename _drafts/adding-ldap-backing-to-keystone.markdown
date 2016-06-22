---
layout: post
title: "Adding LDAP backing to specific domains in Keystone"
---

Introduction
-------------

One of my first tasks here was to set up a working Tripleo installation (which I
did, using [this](http://adam.younglogic.com/2016/05/freeipa-tripleo-undercloud/),
with some modifications) and add an LDAP backed domain to Keystone. I'll be throwing
together a quick post on what modifications I did to make it work for today's releases.

In general, I simply followed my mentor's blog posts to add LDAP backing to a specific
domain. This post is meant to document the places where I deviated from the post.

Quick one-liner on what Keystone is: Keystone is the OpenStack service that handles
the authentication and authorization for the other services, like Glance, Nova, etc.
By default, Keystone is backed by SQL. What if we want a specific domain in Keystone
to be backed by a different driver, like LDAP?

Using Identity API v3 in Keystone
-----------------------------------

Before we start changing configuration values to tell Keystone to use LDAP, we must start
by changing the API version. This is absolutely essential, because Identity v2 does not
support multiple domains. To do that, follow the instructions from
[this post](http://adam.younglogic.com/2016/03/v3fromv2/). In essence, the script changes
the environment variable `OS_IDENTITY_API_VERSION` to be 3, instead of 2.

Enabling domain specific backends
---------------------------------

Enabling domain specific backends is really as simple as editing these values under the
`[identity]` block in `/etc/keystone/keystone.conf`.

```
[identity]
domain_specific_drivers_enabled = true
domain_config_dir = /etc/keystone/domains
```

This is telling Keystone two things: a) that domains can have specific drivers (i.e., one
domain is backed by LDAP, and another is backed by SQL), and b) check under `/etc/keystone/domains`
for any specific domain configuration files.

Now, make a directory called `domains` under `/etc/keystone`. Note that this directory, and all
files in it need to belong to the `keystone` user. Use `chown` to fix your problems, if it isn't
already.

Keystone will look in this directory for any domain specific config files, and they should be of
the format keystone.x.conf, where x is the name of the domain we are configuring.

Follow the posts listed below in Resources for what the config file should look like, but note that
to pick up the changes, we restart httpd instead of openstack-keystone. So:
```
sudo systemctl restart httpd.service
```
Resources
---------
http://adam.younglogic.com/2015/02/adding-an-ldap-backed-domain-to-a-packstack-install/
http://adam.younglogic.com/2014/08/getting-service-users-out-of-ldap/

