---
layout: post
title: "Problematic Ironic Nodes"
---

Introduction
-------------

Last time, we were trying to reinstall the undercloud so that it used a different IP
block. The reinstall succeeded, but now the two Ironic nodes that would normally be
provisioned as a compute node and a control node are in a very strange state.

The problem: both nodes have the power state 'None'.
