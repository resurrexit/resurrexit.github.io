---
layout: post
title: "Debugging LDAP-backed domains in Keystone"
---

Backstory
---------

The story behind this is that I was setting up a system where the FreeIPA[^1] server
is sitting alongside undercloud. The Keystone service for each node in the overcloud
are pointing to this FreeIPA server (let us call this server `identity`). Keystone for
each node is also configured to use
[Identity API v3](http://specs.openstack.org/openstack/keystone-specs/api/v3/identity-api-v3.html),
with a `freeipa` domain backed by identity's LDAP store.

See [this]({% post_url 2016-06-23-adding-ldap-backing-to-keystone %}) for more information
on the configuration this is about.

Problem
-------

I had everything set up manually, the goal being to eventually modify the
[tripleo-heat-templates](https://github.com/openstack/tripleo-heat-templates) so that one can
have this system after running tripleo-quickstart, just by setting the right environment variables.

Of course, something was going wrong. Running `openstack user list --domain freeipa` was not returning
anything, even though I knew I had previously configured Keystone to talk to Identity for all queries
belonging to the `freeipa` domain. *Theoretically*, it should have have returned the users I added by
running `ipa user-add` on Identity.

Debugging
----------

### Step 1: Looking for clues
First, start by looking at `keystone.log`, found in `/var/log/keystone`. This is the log file for the Keystone
service that we want to integrate with FreeIPA. See any anomalous messages?

We can also look at the output from `openstack user list --domain heat_stack`. It should look like this (output
shortened for brevity)
```
+---------+------------------+
| ID      | Name             |
+---------+------------------+
| d5...8c | admin            |
| 42...f8 | aodh             |
| a9...72 | ceilometer       |
| b5...77 | glance           |
| f5...cd | heat             |
+---------+------------------+
```
The `heat_stack` domain is not reliant upon `freeipa`; it should return something like the above. If it doesn't,
your problem is not in your LDAP configuration for Keystone.

#### Minor digression
In fact, if it lags forever without returning -- if all OpenStack services lag without returning, take a closer look
at Keystone. Run `openstack token issue --debug` to examine the kinds of calls it is making. I regret not having
taken notes on the debugging process of this particular problem (that would have been another post in and of itself),
but I was seeing errors like so:
```
Making authentication request to http://192.0.2.1:5000/v2.0/tokens  
Resetting dropped connection: 192.0.2.1
```

In my case, restarting rabbitmq.service worked. It was causing Keystone authentication to fail silently, leading to
all OpenStack services to lag for an eternity (unconfirmed).

```
systemctl restart rabbitmq.service
```

### Step 2: Can we reach our identity server?
There isn't much that's useful here. Nothing that will give us a path to start research on. We can start
by checking to see that running an LDAP query manually works. If it succeeds, we will know that a) identity is
reachable from our node, and b) LDAP queries against identity work, so it's not a problem there. You'll want to
use this:
```
ldapsearch -x -H ldap://warp.lab4.eng.bos.redhat.com -L -b 'dc=warp,dc=lab4,dc=eng,dc=bos,dc=redhat,dc=com' '*'
```

### Step 3: Collecting more information
In my case, the LDAP query succeeded. Good. Now, we need Keystone to dump more information as it runs. To do this,
we will enable `ldap=DEBUG` in `keystone.conf`, which can found under `/etc/keystone`.
Run `openstack user list --domain freeipa` again. Assuming you've done the configuration for the domain correctly,
it should attempt an LDAP query against the server indicated in `keystone.freeipa.conf`. We examine `keystone.log`
again, to see this:
```
2016-06-21 20:20:35.013 1151 DEBUG keystone.common.ldap.core [req-55b9fd81-cc5f-4b29-b237-4e3994d7f0d6 bcdfefb36f44443ca9a8f3cfa445ab40 ec662f250a85453cb40054f3aff49b58 - default default] LDAP search: base=cn=users,cn=accounts,dc=identity,dc=warp,dc=lab4,dc=eng,dc=bos,dc=redhat,dc=com scope=1 filterstr=(&(objectClass=inetOrgPerson)(uid=*)) attrs=['uid', 'userPassword', 'enabled', 'mail', 'description'] attrsonly=0 search_s /usr/lib/python2.7/site-packages/keystone/common/ldap/core.py:938
```

This is what we want. As we can see, the dumped information has the LDAP query that Keystone is running against
identity. Note that the LDAP query style in the logs is not the command we will be running manually.
Compare this
```
LDAP search: base=cn=users,cn=accounts,dc=identity,dc=warp,dc=lab4,dc=eng,dc=bos,dc=redhat,dc=com scope=1 filterstr=(&(objectClass=inetOrgPerson)(uid=*)) attrs=['uid', 'userPassword', 'enabled', 'mail', 'description'] 
```
versus this
```
ldapsearch -x -H ldap://identity.warp.lab4.eng.bos.redhat.com -L -b 'cn=users,cn=accounts,dc=warp,dc=lab4,dc=eng,dc=bos,dc=redhat,dc=com' '(&(objectClass=inetOrgPerson)(uid=*))' 'uid', 'userPassword', 'enabled', 'mail', 'description'
```

### Step 4: Experimenting to find the problem
Convert the LDAP search in the logs to a CLI-style call. Run it, and if it fails, play around with the LDAP query until it works. Look at the format of the kinds of queries that fail and the formats of the kind that succeed. That
should give you pointers to what you have wrong in your `/etc/keystone/domains/keystone.DOMAIN.conf` file. If we've gotten this far, we know that we have the configuration for multiple domains in `/etc/keystone/keystone.conf`
working. Once you've found the problem, update the file and restart httpd to pick up the changes with `systemctl restart httpd.service`. From here, run commands against the FreeIPA domain to make sure Keystone is pointing there.

Keep in mind
------------

The command `openstack user list` behaves oddly when the Identity API version is set to be 3. It gives us the list of all the users
in the 'Default' domain. In API version 2, the very same command will give us a list of all the users. If you want the list of all
the users in a certain domain, you must specify it using the `--domain` option, whether it be the domain ID or the domain name.

In addition to this: do not add users via Keystone and LDAP. I repeat: __do not__ add your users by running
`openstack user create`. Odds are, if you are looking into setting this up, it is because you (or your
users) want your FreeIPA integrated with Keystone for a reason. Whatever it is, users created via `openstack
user create` will not be well formed. FreeIPA is very particular about the way user records are formatted.
Keystone *does not* have knowledge on how to do this in a way that will make FreeIPA happy.

In addition to this, Keystone will be writing to the LDAP database[^2]. This is **wrong**, because the
LDAP store is not managed by Keystone. For this reason, Keystone *should* only have read-only LDAP
backends.

Footnotes
---------
[^1]: FreeIPA is an identity management suite for Linux/Unix systems.
[^2]: I know that LDAP is a protocol and not a database. Don't kill me. I'm using the term liberally to mean the LDAP-style database. In FreeIPA, the implementation we are using is 389 Directory Server, which is backed by Berkeley DB.
