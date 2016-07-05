---
layout: post
title: "Using Tripleo-quickstart for a working OpenStack install"
date: 2016-06-30T14:48:31-04:00
---

This post is meant to record my notes on modifications that I made to the install
process outlined [here](http://adam.younglogic.com/2016/05/freeipa-tripleo-undercloud/).
This is *not* meant to be a standalone tutorial.

We are still applying a patch to quickstart that builds the Identity virtual machine,
but instead of using patch 315749, we are using patch 328373 instead. This change
includes the changes made in 315749, which were to modify the roles to allow for
multiple undercloud nodes. Additionally, it also includes two other patch sets: one to
provision the Identity VM, and one to do the setup.

```
mkdir ~/.quickstart
mkdir ~/devel && cd ~/devel
git clone https://github.com/openstack/tripleo-quickstart
cd tripleo-quickstart
git review -d 328373
$HOME/.quickstart/tripleo-quickstart/quickstart.sh --extra-vars identity_setup=true --retain-inventory warp.lab4.eng.bos.redhat.com
```

Again, if you don't have git review set up, pull and apply the changes described
[here](https://review.openstack.org/#/c/328373/) in a new branch.

For this changeset, you must symlink the playbooks directory and the tripleo-quickstart
directory (both found under ~/.quickstart) to their corresponding directories under the
git repo we just cloned. You may want to do something like:  
```
ln -s $HOME/devel/tripleo-quickstart/playbooks .quickstart/playbooks
```

In addition to this, you may also need to *remove* the directory before creating your symlinks.

Of course, you would substitute in your own FQDN where we have warp.lab4.eng.bos.redhat.com
From here, nothing much changes from the original tutorial, aside from the launch of the
quickstart shell script and the variables in the hosts file. According to the changes in
review.openstack, launch it while passing in the extra variable of `identity_setup=true`.

As mentioned in the previous paragraph,  remember to, edit the `deployment_dir` variable in your
~/.quickstart/hosts file to match your directory. I *suspect* this variable isn't needed, being
that we are not using [ossipee](https://github.com/admiyo/ossipee) to build our FreeIPA server.[^1]
Instead, we are *trying* to use tripleo-quickstart with patch 328373 to do this.

Definitions
-----------

[^1]: Much like tripleo-quickstart, Ossipee is a project meant to build a small OpenStack cluster, using a PackStack install. Ossipee predates tripleo-quickstart. That leads me to PackStack: PackStack is a command line utility using Puppet modules to configure and installs everything via RPMs. PackStack is for existing servers. Compare that with Devstack, which uses git checkouts and shell scripts to install everything needed for OpenStack.
