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
exception of: VIPs/Haproxy, rabbitmq, redis, mongo and galera.

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
* Mongo
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
whole upgrade/update topic, is much easier to manage.

Alternatives
------------

There are many alternative designs for the HA architecture. The decision
to use pacemaker only for a certain set of "core" services and all the
Active/Passive services comes from a careful balance between complexity
of the architecture and its management and being able to recover resources
in a known broken state.

The reason for using only pacemaker for the core services and not, for
example keepalived for the Virtual IPs, is to keep the stack simple and
not introduce multiple distributed resource managers.

The reason for keeping haproxy under pacemaker's management is that 
we can guarantee that a VIP will always run where haproxy is running.


Security Impact
---------------

No changes regarding security aspects compared to the existing status quo.

Other End User Impact
---------------------

The operators working with a cloud are impacted in the following ways:

* The services (galera, redis, mongo, openstack-cinder-volume, VIPs,
  haproxy) will be managed as usual via `pcs`. Pacemaker will monitor these
  services

* All other services will be managed via `systemctl` and systemd will be
  configured to automatically restart a failed service.

* All services will be configured to retry indefinitely to connect to
  the database or to the messaging broker. In case of a controller failure,
  the failover scenario will be the same as with the current HA architecture,
  with the difference that the services will just retry to re-connect indefinitely.


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

Who is leading the writing of the code? Or is this a blueprint where you're
throwing it out there to see who picks it up?

If more than one person is working on the implementation, please designate the
primary author and contact.

Primary assignee:
  michele

Work Items
----------

Work items or tasks -- break the feature up into the things that need to be
done to implement it. Those parts might end up being done by different people,
but we're mostly trying to understand the timeline for implementation.


Dependencies
============

* Include specific references to specs and/or blueprints in tripleo, or in other
  projects, that this one either depends on or is related to.

* If this requires functionality of another project that is not currently used
  by Tripleo (such as the glance v2 API when we previously only required v1),
  document that fact.

* Does this feature require any new library dependencies or code otherwise not
  included in OpenStack? Or does it depend on a specific version of library?


Testing
=======

Please discuss how the change will be tested.

Is this untestable in CI given current limitations (specific hardware /
software configurations available)? If so, are there mitigation plans (3rd
party testing, gate enhancements, etc).


Documentation Impact
====================

What is the impact on the docs? Don't repeat details discussed above, but
please reference them here.


References
==========

Please add any useful references here. You are not required to have any
reference. Moreover, this specification should still make sense when your
references are unavailable. Examples of what you could include are:

* Links to mailing list or IRC discussions

* Links to notes from a summit session

* Links to relevant research, if appropriate

* Related specifications as appropriate (e.g.  if it's an EC2 thing, link the EC2 docs)

* Anything else you feel it is worthwhile to refer to
