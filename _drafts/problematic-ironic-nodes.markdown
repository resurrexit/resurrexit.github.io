---
layout: post
title: "Problematic Ironic Nodes"
---

Introduction
-------------

Last time, we were trying to reinstall the undercloud so that it used a different IP
block. The reinstall succeeded, but now the two Ironic nodes that would normally be
provisioned as a compute node and a control node are in a very strange state.

The problem: both nodes have the power state 'None'. Aside from it being very odd, this
also means that we can't do anything with the nodes as-is. The power state 'None' is a
bit trickier to debug than the other 'wrong' states. There are some tips to help in
debugging Ironic problems.

Run `ironic node-list`. You want the baremetal nodes to be in the following states: Power
State == Off, Provisioning State == available, and Maintenance == False.

Maintenance mode
----------------

Nova will not schedule Ironic nodes that are in maintenance mode. If we want to be able to
schedule a node in maintenance mode, we have to set it to some other state that Ironic
will allow.

It is important to note that in any given state there will be state changes that are not
allowed. See this state machine diagram for more information:
{% img center /assets/images/states.svg 'state machine diagram' %}
[Image source](find the ironic documentation site where this pic is from)

Orphaned nodes
---------------

You may also have orphaned Ironic nodes, where the node is available, but still has an
`instance_uuid` assigned to it. To check if the node is available, run `nova list` and
`heat stack-list`. If the stack is deleted and Nova does not show any servers up and
running, then the baremetal node is still available.

This is something that Nova should do upon deletion, but in the case it failed to,
clear the `instance_uuid` the node is holding onto by running

```
ironic node-update <node_uuid> remove instance_uuid
```

Note: don't do what I did. Do not substitute the actual `instance_uuid` in.

Incorrect SSH credentials
--------------------------

You might be trying to do some operation upon Ironic nodes, but coming across the
error message.

```
Failed to validate power driver interface. Can not delete instance. Error: SSH connection
cannot be established: Failed to establish SSH connection to host 192.168.23.1. (HTTP 500)
```

To see which parameters are missing, run `ironic node-validate <node>`. You might see something
like this:

```
$ ironic node-validate 94066ef2-8604-4041-962b-d68ccc5fdf4a  
+------------+--------+------------------------------------------------------------------------------------------------+  
| Interface  | Result | Reason                                                                                         |  
+------------+--------+------------------------------------------------------------------------------------------------+  
| boot       | True   |                                                                                                |  
| console    | False  | Missing 'ssh_terminal_port' parameter in node's 'driver_info'                                  |  
| deploy     | True   |                                                                                                |  
| inspect    | True   |                                                                                                |  
| management | True   |                                                                                                |  
| power      | False  | SSH connection cannot be established: Failed to establish SSH connection to host 192.168.23.1. |  
| raid       | True   |                                                                                                |  
+------------+--------+------------------------------------------------------------------------------------------------+  
```

The provided SSH credentials are incorrect.
