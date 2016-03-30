..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Pacemaker Lightweight Architecture
==================================

https://blueprints.launchpad.net/tripleo/+spec/pacemaker-lightweight-architecture

Create a template that deploys a minimal pacemaker architecture, where
all the openstack services are started and monitored by systemd with the
exception of: VIPs/Haproxy, rabbitmq, redis and galera.

Problem Description
===================

The pacemaker architeture deployed currently via the pacemaker templates
manages every single service on the controllers via pacemaker. This approach,
while having the advantage of having a single entity managing and monitoring
all services, does bring a certain complexity to it and assumes that the
operaters are quite familiar with pacemaker and the management of resources
with it. The aim is to create a new template that deploys controllers
where pacemaker controls the following resources:

* Virtual IPs + HAProxy
* RabbitMQ
* Galera
* Redis
* openstack-cinder-volume (as the service is not A/A yet)

Proposed Change
===============

Overview
--------

Initially, the plan is to create a brand new template called
overcloud_controller_pacemaker_lightweight.yaml which will enable most
services via systemd and which will remove most constraints.
In terms of ordering constraints we will go from a graph like this one:
http://acksyn.org/files/tripleo/wsgi-openstack-core.pdf (mitaka)

to a graph like this one:
http://acksyn.org/files/tripleo/light-cib.pdf (lighweight-mitaka)

Once this new architecture is in place and we have tested it extensively,
we can bless it as the new only HA architecture and also work on the upgrade
path from the previous fully-fledged pacemaker HA architecture to this
new lightweight one. The end goal is for the lightweight architecture
to supplant the existing HA one. Since the impact of pacemaker in the
lightweight architecture is quite small, it is possible to consider
dropping the non-ha architecture and using only lightweight for every
deployment and every CI job. The decision on that will depend a bit
on how many corner cases are found.

Another side-benefit is that with this lightweight architecture the
whole upgrade/update topic, is much easier to manage with TripleO.

Alternatives
------------

There are many alternative designs for the HA architecture. The decision
to use pacemaker only for a certain set of "core" services and all the
Active/Passive services comes from a careful balance between complexity
of the architecture and its management and being able to recover resources
in a known broken state. There is a main assumption here about native
openstack services:
They *must* be able to start with the broker and the database down and keep
retrying.

The reason for using only pacemaker for the core services and not, for
example keepalived for the Virtual IPs, is to keep the stack simple and
not introduce multiple distributed resource managers. Also, if we used
only keepalived, we'd have no way of recovering from a failure beyond
trying to relocate the VIP.

The reason for keeping haproxy under pacemaker's management is that 
we can guarantee that a VIP will always run where haproxy is running,
should an haproxy service start failing.


Security Impact
---------------

No changes regarding security aspects compared to the existing status quo.

Other End User Impact
---------------------

The operators working with a cloud are impacted in the following ways:

* The services (galera, redis, openstack-cinder-volume, VIPs,
  haproxy) will be managed as usual via `pcs`. Pacemaker will monitor these
  services

* All other services will be managed via `systemctl` and systemd will be
  configured to automatically restart a failed service.

* All services will be configured to retry indefinitely to connect to
  the database or to the messaging broker. In case of a controller failure,
  the failover scenario will be the same as with the current HA architecture,
  with the difference that the services will just retry to re-connect indefinitely.

* Previously with the HA template every service would be monitored and managed by
  pacemaker. With the split between openstack services being managed by systemd and
  "core" services managed by pacemaker, the operator needs to know which service
  to monitor with which command.

Performance Impact
------------------

No changes compared to the existing architecture.

Other Deployer Impact
---------------------

Discuss things that will affect how you deploy and configure OpenStack
that have not already been mentioned, such as:

* Until we switch the HA architecture to be lightweight we need to maintain
  a lightweight job in CI

Developer Impact
----------------

There is an additional template until Newton+1, which will need to be maintained.
After Newton, it is entirely possible that we use this single lightweight
template for both the HA and the non-HA deployments.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  michele

Other contributors:
  ...


Work Items
----------

* Prepare the template that deploys the lightweight architecture.
  Initially, keep it as close as possible to the existing HA template and
  make it simpler in a second iteration (remove unnecesary steps, etc.)
  Template currently lives here and deploys successfully: 
    
    https://github.com/mbaldessari/tripleo-heat-templates/tree/wip-mitaka-lightweight-arch

* Test failure scenarios and recovery scenario, open bugs against services
  that misbehave in the face of database and/or broker being down


Dependencies
============

None

Testing
=======

So initial smoke-testing has been completed successfully. Another set of
tests focusing on the behaviour of openstack services when galera and rabbitmq
are down is in the process of being run. 

Particular focus will be on failover scenarios and recovery times and making
sure that there are no regressions compared to the current HA architecture.


Documentation Impact
====================

Currently we do not describe the architectures as deployed by TripleO itself,
so no changes needed. A short page in the docs describing the arch would be a nice
thing to have in the future.

References
==========

This design came mostly out from a meeting in Brno with the following attendees:

* Chris Feist
* Fabio Di Nitto
* Javier Peña
* Lars Kellogg-Steadman
* Mark Mcloughlin
* Michele Baldessari
* Raoul Scarazzini
* Rob Young
