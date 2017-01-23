.. _multi-regions:

======================================
Multiple Regions Deployment with Kolla
======================================

This section describes how to perform a basic multiple regions deployment
with Kolla. A basic multiple regions deployment consists in separate
OpenStack installation in two or more regions (RegionOne, RegionTwo, ...)
with a shared Keystone and Horizon. The following assumes Keystone and
Horizon are deployed in RegionOne, and other regions have access to the
admin endpoint (i.e., ``kolla_internal_fqdn``) of RegionOne. It also
supposed that the operator knows the name of all OpenStack regions in
advance, and considers as many Kolla deployment as there are regions.

Deployment of the first region with Keystone and Horizon
========================================================

Deployment of the first region results in a typical Kolla deployment
whenever, it is an *all-in-one* or *multinode* deployment (see
:ref:`deploying-kolla`). It only requires slight modifications in the
``globals.yml`` configuration file. First of all, ensure that Keystone and
Horizon are enabled:

::

   enable_keystone: "yes"
   enable_horizon: "yes"

Then, change the value of ``multiple_regions_names`` to add names of other
regions. In this example, we consider two regions. The current one,
formerly knows as RegionOne, that is hided behind
``openstack_region_name`` variable, and the RegionTwo:

::

   openstack_region_name: "RegionOne"
   multiple_regions_names:
       - "{{ openstack_region_name }}"
       - "RegionTwo"

.. note:: Kolla uses these variables to create necessary endpoints into
          Keystone so that services of other regions can access it. Kolla
          also updates the Horizon ``local_settings`` to support multiple
          regions.

Finally, note the value of ``kolla_internal_fqdn`` and run
``kolla-ansible``. The ``kolla_internal_fqdn`` value will be used by other
regions to contact Keystone. For the sake of this example, we assume the
value of ``kolla_internal_fqdn`` is ``10.10.10.254``.

Deployment of other regions
===========================

Deployment of other regions follows an usual Kolla deployment except that
OpenStack services connect to the RegionOne's Keystone. This implies to
update the ``globals.yml`` configuration file to tell Kolla how to reach
Keystone. In the following, ``kolla_internal_fqdn_r1`` refers to the value
of ``kolla_internal_fqdn`` in RegionOne:

::

   kolla_internal_fqdn_r1: 10.10.10.254

   keystone_admin_url: "{{ admin_protocol }}://{{ kolla_internal_fqdn_r1 }}:{{ keystone_admin_port }}/v3"
   keystone_internal_url: "{{ internal_protocol }}://{{ kolla_internal_fqdn_r1 }}:{{ keystone_public_port }}/v3"

   openstack_auth:
       auth_url: "{{ admin_protocol }}://{{ kolla_internal_fqdn_r1 }}:{{ keystone_admin_port }}"
       username: "admin"
       password: "{{ keystone_admin_password }}"
       project_name: "admin"

Configuration files of nova/neutron/glance... also have to be updated to
contact RegionOne's Keystone. Hopefully, Kolla offers to override all
configuration files at the same time thanks to the ``node_custom_config``
variable (see :ref:`service-config`). This implies to create a
``global.conf`` file with the following content:

::

   [keystone_authtoken]
   auth_uri = {{ keystone_internal_url }}
   auth_url = {{ keystone_admin_url }}

and link this file into the ``globals.yml``:

::

   node_custom_config: path/to/the/global.conf

Also, change the name of the current region. For instance, RegionTwo:

::

   openstack_region_name: "RegionTwo"

Finally, disable the deployment of Keystone and Horizon that are
unnecessary in this region and run ``kolla-ansible``:

::

   enable_keystone: "no"
   enable_horizon: "no"

The configuration is the same for any other region.
