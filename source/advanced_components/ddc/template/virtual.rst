.. _ddc_virtual:

============================
Virtual Objects Provisioning
============================

With OneProvision you can grow up the physical resources available in your cloud. Once you have your infrastructure ready, you need to create virtual objects
in it, in order to use it. In this section we are going to show a way to deploy everything at once with OneProvision.

We are going to deploy a cluster with the following elements:

- 1 Host
- 2 Datastores
- 1 Virtual Network
- 1 Virtual Machine Template
- 1 Image

.. code::

    ---
    playbook: "static_vxlan"
    name: "packet_cluster"

    defaults:
    provision:
        provider: "packet"
        packet_token: "***************************"
        packet_project: "*************************"
        facility: "ams1"
        plan: "baremetal_0"
        os: "centos7"
    connection:
        public_key: '/var/lib/one/.ssh/id_rsa.pub'
        private_key: '/var/lib/one/.ssh/id_rsa'
    configuration:
        opennebula_node_kvm_param_nested: true

    cluster:
    name: "packet_cluster"

    hosts:
      - reserved_cpu: 100
        im_mad: kvm
        vm_mad: kvm
        provision:
          hostname: "<%= @name %>-host"

    datastores:
      - name: "<%= @name %>-images"
        ds_mad: fs
        tm_mad: ssh
      - name: "<%= @name %>-system"
        type: system_ds
        tm_mad: ssh
        safe_dirs: "/var/tmp /tmp"

    networks:
      - name: "<%= @name %>-hostonly_nat"
        vn_mad: dummy
        bridge: br0
        dns: "8.8.8.8 8.8.4.4"
        gateway: "192.168.150.1"
        description: "Host-only networking with NAT"
        filter_ip_spoofing: "YES"
        filter_mac_spoofing: "YES"
        ar:
          - ip: 192.168.150.2
            size: 253
            type: IP4

    marketplaceapps:
      - appname: "Ttylinux - KVM"
        name: "TTY"
        dsid: <%= @body['provision']['infrastructure']['datastores'][0]['id'] %>
        meta:
          wait: true
          wait_timeout: 30

.. prompt:: bash $ auto

    $ oneprovision validate virtual.yaml && echo OK
    OK

We can now deploy the template using the command ``oneprovision create``:

.. prompt:: bash $ auto

    $ oneprovision create virtual.yaml
    ID: 24959b5c-8eca-4cd8-a3bb-dac36a7b5c1d

This will deploy the host in Packet, will configure it and will create all the objects in OpenNebula. With this specific part:

.. code::

    marketplaceapps:
    - appname: "Ttylinux - KVM"
      name: "TTY"
      dsid: <%= @body['provision']['infrastructure']['datastores'][0]['id'] %>
      meta:
        wait: true
        wait_timeout: 30

OneProvision is going to export the application called Ttylinux - KVM from the marketplace into OpenNebula.
It will store the image in the image datastores created in this provision and will wait until the image is in **ready** state.

The final result would be the following:

.. prompt:: bash $ auto

    $ oneprovision show 18 -x
    <DOCUMENT>
    <ID>18</ID>
    <UID>0</UID>
    <GID>0</GID>
    <UNAME>oneadmin</UNAME>
    <GNAME>oneadmin</GNAME>
    <NAME>testing</NAME>
    <TYPE>103</TYPE>
    <PERMISSIONS>
        <OWNER_U>1</OWNER_U>
        <OWNER_M>1</OWNER_M>
        <OWNER_A>0</OWNER_A>
        <GROUP_U>0</GROUP_U>
        <GROUP_M>0</GROUP_M>
        <GROUP_A>0</GROUP_A>
        <OTHER_U>0</OTHER_U>
        <OTHER_M>0</OTHER_M>
        <OTHER_A>0</OTHER_A>
    </PERMISSIONS>
    <TEMPLATE>
        <BODY><![CDATA[{"name":"testing","description":null,"start_time":1600684065,"state":3,"provider":"packet","provision":{"infrastructure":{"datastores":[{"name":"tf-images","id":128},{"name":"tf-system","id":129}],"networks":[{"name":"tf-hostonly_nat","id":14}],"hosts":[{"name":"provision-cbbe1e477a1bd5e1324ae66bdffc20e28ae0b0b93f10db43","id":"18"}],"clusters":[{"name":"tf","id":114}]},"resource":{"images":[{"id":18,"name":"test_image"}],"templates":[{"id":15,"name":"test_template"}],"vntemplates":[{"id":11,"name":"vntemplate"}],"flowtemplates":[{"id":19,"name":"my_service"}]}}}]]></BODY>
    </TEMPLATE>
    </DOCUMENT>

As you can see all the objects have been created and they belong to the same provision. This means, that when you for example delete the provision
all the objects are going to be deleted as once.

In the next sections you can check what objects can be created with oneprovision and also more details about the different options that are available.

.. _ddc_virtual_objects:

Managing Virtual Objects
========================

In this section you can check how to create all the virtual objects you need to have a cluster ready to instantiate a virtual machine. These objects are
created with the provision itselt, with the command ``oneprovision create`` and can be deleted at once with the command ``oneprovision delete``.

The available objects are the following:

- Images
- Marketplace apps
- Virtual Machine Templates
- Virtual Network Templates
- OneFlow Service templates

.. note:: The marketplace apps are exported from the marketplace, so they create a new imagen and a new template.

.. note:: They are created in the order they appear in the list.

Virtual Machine Templates
-------------------------

To create virtual machine templates, you need to add the following to your provision template:

.. code::

    templates:
      - name: "test_template"
        memory: 1
        cpu: 1

These are the three mandatory values for a template in OpenNebula. Then you can add all the information you want, everything you put there will be copy
to the virtual machine template. Please refer to this :ref:`guide<template>` to know what attributes are available.

Virtual Network Templates
-------------------------

To create virtual machine templates you need to add the following to your provision template:

.. code::

    vntemplates:
      - name: "test_vntemplate"
        vn_mad: "bridge"
        ar:
          - ip: "10.0.0.1"
            size: 10
            type: "IP4"

These are the mandatory values for a virtual network template in OpenNebula. Then you can add all the information you want, everything you put there will be copy
to the virtual network template. Please refer to this :ref:`guide<vn_templates>` to know what attributes are available.

Images
------

To create images you need to add the following to your provision template:

.. code::

    images:
      - name: "test_image"
        ds_id: 1
        size: 2048

These are the three mandatory values for an image in OpenNebula. Then you can add all the information you want, everything you put there will be copy
to the image template. Please refer to this :ref:`guide<img_template>` to know what attributes are available.

Marketplace Apps
----------------

In this case, the marketplace app is not created, but exported from marketplace. To do this you need to add the following to your provision template:

.. code::

    marketplaceapps:
      - appid: 238
        name: "test_image_2"
        dsid: 1

These are the three mandatory values to export an image from the marketplace. Please refer to this :ref:`guide<marketapp>` to know what more options are available.

.. note:: You can also use **appname** instead of appid.

OneFlow Service Templates
-------------------------

To create a service template you need to add the following to your provision template:

.. code::

    flowtemplates:
      - name: "test_service"
        deployment: "straight"
        roles:
          - name: "frontend"
            vm_template: 0
          - name: "backend"
            vm_template: 1

These are the mandatory values for a service template. Please refer to this :ref:`guide<appflow_use_cli>` to know more about OneFlow templates.

.. note:: You can create more than one object at once, just add more elements to the specific list.

.. _ddc_virtual_perms:

Ownership and Permissions
=========================

All the virtual OpenNebula objects are created by oneprovision itself, by default the ownership correspond to the user executing the tool, normally it is oneadmin. In case
you want to change the ownership or permissions you can add the following attributes to the template:

- **uid**: user ID for the object.
- **gid**: groupd ID for the object.
- **uname**: user name for the object.
- **gname**: group name for the object.
- **mode**: permissions in octet format for the object.

For example, if we want to change the user for an specific image, we should add the following:

.. code::

    images:
        - name: "test_image"
          ds_id: 1
          size: 2048
          meta:
            uid: 1

In this example, the image owner after creation finished would be serveradmin, which is the user with ID 1.

This applies to all objects and you can combine the three of them, for example:

.. code::

    images:
        - name: "test_image"
          ds_id: 1
          size: 2048
          meta:
            uid: 1
            gid: 1
            mode: 644

In this example, the image owner would be serveradmin, the group would be users and the permissions would be 644.

You can also use the **uname** and **gname**, for example:

.. code::

    images:
        - name: "test_image"
          ds_id: 1
          size: 2048
          meta:
            uname: user1
            gname: users
            mode: 644

In this example, the image owner would be user1 and the group would be users.

.. _ddc_virtual_wait:

Wait Modes
==========

Some objects take a bit to be ready, concretely images depending on the size. To manage this, you can use the attribute wait, it can have two possible values:

- **false**: just create the objects and continue.
- **true**: create objects and wait until they are successfully imported.

Theses wait modes are also combined with :ref:`run modes <ddc_usage>`. So if the object fails when waiting to it, the tool is going to check waht run mode needs to apply.

For example:

.. code::

    images:
      - name: "test_image"
        ds_id: 1
        size: 2048
        meta:
          wait: false
          mode: 644

In this example, the image would be created and there will not be any wait until it is ready, the program would continue.

The timeout to wait until the resource is ready is also configurable, it can be done adding **wait_timeout** attribute in the object. For example:

.. code::

    images:
      - name: "test_image"
        ds_id: 1
        size: 2048
        meta:
          wait: true
          wait_timeout: 30

In this example, the timeout to wait would be 30 seconds.

.. warning:: Wait attribute is only available for images and marketplace apps.

Using Wait Globally
-------------------

As we have seen, you can set the wait per object in the provision template, but you can also set it globally using the CLI. There are two parameters available:

- **wait-ready**: with this the tool will wait until the resources are ready.
- **wait-timeout timeout**: with this you can set the timeout (default = 60s).

.. note:: The provision template wait and timeout are not overwritten by these parameters in the command, if you set some in the template they are respected.

For example:

.. code::

    $ oneprovision create virtual.yaml --wait-ready --wait-timeout 60

With this command the program will wait for all objects with a timeout of 60 seconds.

.. _ddc_virtual_all:

Referencing Objects
===================

As all these are objects that are created dynamically, there can be some relations between them. For example, we might want to use a new image that is created
in a template that is going to be created too.

For this, ERB expressions are available, so you can reference objects that have been already created. Let's see some example about this.

.. code::

    datastores:
      - name: "test_images"
        ds_mad: fs
        tm_mad: ssh
      - name: "test_system"
        type: system_ds
        tm_mad: ssh
        safe_dirs: "/var/tmp /tmp"

    images:
      - name: "test_image"
        ds_id: <%= @body['provision']['infrastructure']['datastores'][0]['id'] %>
        size: 2048

In this example, we create two datastores (system and images) and an image. We want to store the image in the image datastore we just created, so we can
reference it using the ERB expression ``@body['provision']['infrastructure']['datastores'][0]['id']``.

.. code::

    images:
      - name: "test_image"
        ds_id: 1
        size: 2048

    templates:
      - name: "test_template"
        memory: 1
        cpu: 1
        disk:
          - image_id: <%= @body['provision']['resource']['images'][0]['id'] %>

In this example, we create an image and a template. We want the template to have a disk referencing to the new image, so we can reference it using
the ERB expression ``@body['provision']['resource']['images'][0]['id']``.

.. warning:: The order of objects creation is the following:

    - Images
    - Marketplace apps
    - Templates
    - VNetTemplates
    - Service templates

Full Example
------------

Here you can check a full provision template example:

.. code::

    name: myprovision
    playbook: default

    # Global defaults:
    defaults:
    provision:
      provider: packet
      packet_token: **************
      packet_project: ************
      facility: ams1
      plan: baremetal_0
      os: centos_7
    connection:
      public_key: '/var/lib/one/.ssh/id_rsa.pub'
      private_key: '/var/lib/one/.ssh/id_rsa'
    configuration:
      opennebula_node_kvm_param_nested: true

    # List of OpenNebula infrastructure objects to deploy with provision/connection/configuration overrides
    cluster:
    name: mycluster

    hosts:
      - reserved_cpu: 100
        im_mad: kvm
        vm_mad: kvm
        provision:
          hostname: "myhost1"
      - reserved_cpu: 100
        im_mad: kvm
        vm_mad: kvm
        provision:
          hostname: "myhost2"
          os: ubuntu18_04
        connection:
          remote_user: ubuntu

    datastores:
      - name: "myprovision-images"
        ds_mad: fs
        tm_mad: ssh
      - name: "myprovision-system"
        type: system_ds
        tm_mad: ssh
        safe_dirs: "/var/tmp /tmp"

    networks:
      - name: "myprovision-hostonly_nat"
        vn_mad: dummy
        bridge: br0
        dns: "8.8.8.8 8.8.4.4"
        gateway: "192.168.150.1"
        description: "Host-only networking with NAT"
        filter_ip_spoofing: "YES"
        filter_mac_spoofing: "YES"
        ar:
          - ip: 192.168.150.2
            size: 253
            type: IP4

    images:
      - name: "test_image"
        ds_id: <%= @body['provision']['infrastructure']['datastores'][0]['id'] %>
        size: 2048
        meta:
          uid: 1
          gid: 100
          mode: 644

    marketplaceapps:
      - appid: 238
        name: "test_image2"
        dsid: <%= @body['provision']['infrastructure']['datastores'][0]['id'] %>
        meta:
          uid: 1
          gid: 100
          mode: 600
          wait: true

    templates:
      - name: "test_template"
        memory: 1
        cpu: 1
        disk:
          - image_id: <%= @body['provision']['resource']['images'][1]['id'] %>
        nic:
          - network_id: <%= @body['provision']['infrastructure']['networks'][0]['id'] %>
        meta:
          uid: 1
          gid: 100
          mode: 777

    vntemplates:
      - name: "vntemplate"
        vn_mad: "bridge"
        ar:
          - ip: "10.0.0.1"
            size: 10
            type: "IP4"
        cluster_ids: <%= @body['provision']['infrastructure']['clusters'][0]['id'] %>

    flowtemplates:
      - name: "my_service"
        deployment: "straight"
        roles:
          - name: "frontend"
            vm_template: <%= @body['provision']['resource']['templates'][0]['id'] %>
        meta:
          uid: 1
          gid: 100
