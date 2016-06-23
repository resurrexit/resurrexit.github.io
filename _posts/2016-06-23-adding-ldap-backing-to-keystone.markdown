---
layout: post
title: "Adding LDAP backing to specific domains in Keystone"
date: 2016-06-23T16:29:08-04:00
---

Introduction
-------------

My first task here was to set up a working Tripleo installation (which I
did, using [this](http://adam.younglogic.com/2016/05/freeipa-tripleo-undercloud/),
with some modifications). Next, add an LDAP backed domain to Keystone.

In general, I followed blog posts to add LDAP backing to a domain I created named freeipa.
This is meant to document where I deviated from the posts I followed.

Quick one-liner on what Keystone is: Keystone is the OpenStack service that handles
the authentication and authorization for the other services, like Glance, Nova, etc.
By default, Keystone is backed by SQL. What if we want a specific domain in Keystone
to be backed by a different driver, like LDAP?

Using Identity API v3 in Keystone
-----------------------------------

Before we start changing configuration values to tell Keystone to use LDAP, we must start
by changing the API version. This is essential, because Identity v2 does not support multiple
domains. To do that, use the script from
[this post](http://adam.younglogic.com/2016/03/v3fromv2/). The script changes
the environment variable `OS_IDENTITY_API_VERSION` in your rc files to be 3, instead of 2.

Enabling domain specific backends
---------------------------------

Edit the values in the `[identity]` block indicated below to enable domain specific backends in
`/etc/keystone/keystone.conf`.

```
[identity]
domain_specific_drivers_enabled = true
domain_config_dir = /etc/keystone/domains
```

This is telling Keystone two things:  
  1. domains can have specific drivers (i.e., one domain backed by LDAP and another backed by SQL)  
  2. check under `/etc/keystone/domains` for any specific domain configuration files.  

Now, make a directory `domains` under `/etc/keystone`. Note that this directory, and all
files in it need to belong to the `keystone` user. Use `chown` to fix any issues.

Keystone will look in this directory for any domain specific config files, and they should be of
the format keystone.x.conf, where x is the name of the domain we are configuring.

See the posts below in Resources for what the config file should look like. After you are finished
editing your domain specific config file,  we must restart httpd (instead of openstack-keystone)
to pick up the changes. Use  `sudo systemctl restart httpd.service`.

Resources
---------
[Adding an LDAP backed domain to a Packstack install](http://adam.younglogic.com/2015/02/adding-an-ldap-backed-domain-to-a-packstack-install/)  
[Getting service users out of LDAP](http://adam.younglogic.com/2014/08/getting-service-users-out-of-ldap/)

