---
layout: post
title: "Bug Hunting: Overcloud Edition"
---

Introduction
------------

I've been working on getting a simple overcloud deploy to work for the longest time. At some
point in time, something broke. I haven't been able to fix the failed overcloud deploy for
a while now. In fact, the list of approaches I took to fix it even include tearing down the
entire deploy and recreating it via tripleo-quickstart. Unfortunately, all the approaches
I took until now did not work. So, I decided to dive in deep into the failure I was getting.

The failure
-----------

The failure log was being dumped in $HOME, in the form of `failed_deployment_<uuid>.log` It
will give us ideas of what went wrong. Mine looks like
[this](https://paste.fedoraproject.org/405060/73397147/). In my case, I used `jq` and some
`sed` to clean it up. Something like

```
cat failed_deployment_d32355df-2081-46f8-900a-707898b06355.log | jq '.output_values | .deploy_stderr' | sed 's!\\n!\'$'\n!g'
```

might be useful. In my case, upon further reading the output from `deploy_stdout`, I find lines like
this:

```
Error: Could not prefetch keystone_tenant provider 'openstack': Execution of '/usr/bin/openstack project list --quiet --format csv --long' returned 1: Unable to establish connection to http://192.0.2.7:35357/v3/projects (tried 10, for a total of 170 seconds)
Warning: /Stage[main]/Keystone::Roles::Admin/Keystone_tenant[service]: Skipping because of failed dependencies
```

It seems that Keystone is unavailable or down. Let's take a closer look.
```
nova list  
ssh heat-admin@<controller-ip-address>  
netstat -tulpn | grep 5000  
```

That confirms it for me. I don't see Keystone listening at either of the 'default' Keystone ports 5000 or
35357. The next step would be to try to restart it.

```
systemctl status
systemctl restart httpd.service
```

After a long time, a timeout error message is printed. 
```
Execution of '/usr/bin/systemctl start httpd' returned 1: Job for httpd.service failed because a timeout was exceeded. See \"systemctl status httpd.service\" and \"journalctl -xe\" for details.
```


