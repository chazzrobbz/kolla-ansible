.. _ironic-guide:

================================
Ironic - Bare Metal provisioning
================================

Overview
~~~~~~~~
Ironic works well in Kolla, though it is not thoroughly tested as part of Kolla
CI, so may be subject to instability.

Pre-deployment Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Enable Ironic in ``/etc/kolla/globals.yml``:

.. code-block:: yaml

   enable_ironic: "yes"

In the same file, define a network interface as the default NIC for dnsmasq and
a range of IP addresses that will be available for use by Ironic inspector.
The optional netmask of the network should be provided in case when DHCP-relay
is used. Finally, define a network to be used for the Ironic cleaning network:

.. code-block:: yaml

   ironic_dnsmasq_interface: "eth1"
   ironic_dnsmasq_dhcp_range: "192.168.5.100,192.168.5.110,255.255.255.0"
   ironic_cleaning_network: "public1"

In the same file, optionally a default gateway to be used for the Ironic
Inspector inspection network:

.. code-block:: yaml

   ironic_dnsmasq_default_gateway: 192.168.5.1

In the same file, specify the PXE bootloader file for Ironic Inspector. The
file is relative to the ``/tftpboot`` directory. The default is ``pxelinux.0``,
and should be correct for x86 systems. Other platforms may require a different
value, for example aarch64 on Debian requires
``debian-installer/arm64/bootnetaa64.efi``.

.. code-block:: yaml

   ironic_dnsmasq_boot_file: pxelinux.0

Ironic inspector also requires a deploy kernel and ramdisk to be placed in
``/etc/kolla/config/ironic/``. The following example uses coreos which is
commonly used in Ironic deployments, though any compatible kernel/ramdisk may
be used:

.. code-block:: console

   $ curl https://tarballs.opendev.org/openstack/ironic-python-agent/dib/files/ipa-centos8-master.kernel \
     -o /etc/kolla/config/ironic/ironic-agent.kernel

   $ curl https://tarballs.opendev.org/openstack/ironic-python-agent/dib/files/ipa-centos8-master.initramfs \
     -o /etc/kolla/config/ironic/ironic-agent.initramfs

You may optionally pass extra kernel parameters to the inspection kernel using:

.. code-block:: yaml

   ironic_inspector_kernel_cmdline_extras: ['ipa-lldp-timeout=90.0', 'ipa-collect-lldp=1']

in ``/etc/kolla/globals.yml``.

Configure iPXE HTTP server port (optional)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The port used for the iPXE webserver is controlled via ``ironic_ipxe_port`` in
``/etc/kolla/globals.yml``:

.. code-block:: yaml

    ironic_ipxe_port: "8089"

Revert to plain PXE (not recommended)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Starting with Yoga, Ironic has changed the default PXE from plain PXE to iPXE.
Kolla Ansible follows this upstream decision but allows users to revert to
plain PXE. Please note Kolla Ansible does not support plain PXE and iPXE at the
same time - the user must choose one.

If you have to revert to plain iPXE, set:

.. code-block:: yaml

   enable_ironic_ipxe: "no"

And also remove ``ipxe`` from the ``enabled_boot_interfaces`` in
``/etc/kolla/config/ironic.conf``, leaving only ``pxe`` (and possibly other
alternatives) around:

.. code-block:: yaml

   [DEFAULT]
   enabled_boot_interfaces = pxe

When iPXE booting is enabled, the ``ironic_ipxe`` container is used to serve
the iPXE boot images as described below. Regardless of that setting, the
same container is used to support the ``direct`` deploy interface.

Attach ironic to external keystone (optional)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
In :kolla-ansible-doc:`multi-regional <user/multi-regions.html>` deployment
keystone could be installed in one region (let's say region 1) and ironic -
in another region (let's say region 2). In this case we don't install keystone
together with ironic in region 2, but have to configure ironic to connect to
existing keystone in region 1. To deploy ironic in this way we have to set
variable ``enable_keystone`` to ``"no"``.

.. code-block:: yaml

    enable_keystone: "no"

It will prevent keystone from being installed in region 2.

To add keystone-related sections in ironic.conf, it is also needed to set
variable ``ironic_enable_keystone_integration`` to ``"yes"``

.. code-block:: yaml

    ironic_enable_keystone_integration: "yes"

Deployment
~~~~~~~~~~
Run the deploy as usual:

.. code-block:: console

  $ kolla-ansible deploy


Post-deployment configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The :ironic-doc:`Ironic documentation <install/configure-glance-images>`
describes how to create the deploy kernel and ramdisk and register them with
Glance. In this example we're reusing the same images that were fetched for the
Inspector:

.. code-block:: console

  openstack image create --disk-format aki --container-format aki --public \
    --file /etc/kolla/config/ironic/ironic-agent.kernel deploy-vmlinuz

  openstack image create --disk-format ari --container-format ari --public \
    --file /etc/kolla/config/ironic/ironic-agent.initramfs deploy-initrd

The :ironic-doc:`Ironic documentation <install/configure-nova-flavors>`
describes how to create Nova flavors for bare metal.  For example:

.. code-block:: console

  openstack flavor create --ram 512 --disk 1 --vcpus 1 my-baremetal-flavor
  openstack flavor set my-baremetal-flavor --property \
    resources:CUSTOM_BAREMETAL_RESOURCE_CLASS=1 \
    resources:resources:VCPU=0 \
    resources:resources:MEMORY_MB=0 \
    resources:resources:DISK_GB=0

The :ironic-doc:`Ironic documentation <install/enrollment>` describes how to
enroll baremetal nodes and ports.  In the following example ensure to
substitute correct values for the kernel, ramdisk, and MAC address for your
baremetal node.

.. code-block:: console

  openstack baremetal node create --driver ipmi --name baremetal-node \
    --driver-info ipmi_port=6230 --driver-info ipmi_username=admin \
    --driver-info ipmi_password=password \
    --driver-info ipmi_address=192.168.5.1 \
    --resource-class baremetal-resource-class --property cpus=1 \
    --property memory_mb=512 --property local_gb=1 \
    --property cpu_arch=x86_64 \
    --driver-info deploy_kernel=15f3c95f-d778-43ad-8e3e-9357be09ca3d \
    --driver-info deploy_ramdisk=9b1e1ced-d84d-440a-b681-39c216f24121

  openstack baremetal port create 52:54:00:ff:15:55 \
    --node 57aa574a-5fea-4468-afcf-e2551d464412 \
    --physical-network physnet1

Make the baremetal node available to nova:

.. code-block:: console

  openstack baremetal node manage 57aa574a-5fea-4468-afcf-e2551d464412
  openstack baremetal node provide 57aa574a-5fea-4468-afcf-e2551d464412

It may take some time for the node to become available for scheduling in nova.
Use the following commands to wait for the resources to become available:

.. code-block:: console

  openstack hypervisor stats show
  openstack hypervisor show 57aa574a-5fea-4468-afcf-e2551d464412

Booting the baremetal
~~~~~~~~~~~~~~~~~~~~~
Assuming you have followed the examples above and created the demo resources
as shown in the :doc:`../../user/quickstart`, you can now use the following
example command to boot the baremetal instance:

.. code-block:: console

  openstack server create --image cirros --flavor my-baremetal-flavor \
    --key-name mykey --network public1 demo1

In other cases you will need to adapt the command to match your environment.

Notes
~~~~~

Debugging DHCP
--------------
The following `tcpdump` command can be useful when debugging why dhcp
requests may not be hitting various pieces of the process:

.. code-block:: console

  tcpdump -i <interface> port 67 or port 68 or port 69 -e -n

Configuring the Web Console
---------------------------
Configuration based off upstream :ironic-doc:`Node web console
<admin/console.html#node-web-console>`.

Serial speed must be the same as the serial configuration in the BIOS settings.
Default value: 115200bps, 8bit, non-parity.If you have different serial speed.

Set ironic_console_serial_speed in ``/etc/kolla/globals.yml``:

.. code-block:: yaml

   ironic_console_serial_speed: 9600n8

Deploying using virtual baremetal (vbmc + libvirt)
--------------------------------------------------
See https://brk3.github.io/post/kolla-ironic-libvirt/
