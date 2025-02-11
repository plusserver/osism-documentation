.. _configuration-environment-openstack-nova:

====
Nova
====

.. contents::
   :depth: 2

Local LVM2 storage
==================

* A volume group with the name ``nova`` is created first with file ``environments/kolla/files/overlays/nova/nova-compute.conf``.

.. code-block:: ini

   [libvirt]
   images_type = lvm
   images_volume_group = nova
   sparse_logical_volumes = False
   disk_cachemodes = "file=directsync,block=directsync,network=directsync"

Virtual GPUs
============

* https://docs.openstack.org/nova/latest/admin/virtual-gpu.html

NVIDIA
------

* https://docs.nvidia.com/grid/latest/grid-vgpu-release-notes-generic-linux-kvm/index.html

* identify the available GPUs

.. code-block:: console

   $ lspci | grep NVIDIA
   02:00.0 3D controller: NVIDIA Corporation Device 1b38 (rev a1)
   82:00.0 3D controller: NVIDIA Corporation Device 1b38 (rev a1)
   85:00.0 3D controller: NVIDIA Corporation Device 1b38 (rev a1)
   86:00.0 3D controller: NVIDIA Corporation Device 1b38 (rev a1)

   $ lshw -numeric -C display
   WARNING: you should run this program as super-user.
     *-display
          description: 3D controller
          product: NVIDIA Corporation [10DE:1B38]
          vendor: NVIDIA Corporation [10DE]
   [...]

* resolve the PCI device ID e.g. with https://devicehunt.com

.. image:: /images/devicehunt-nvidia-p40.png

* check https://docs.nvidia.com/grid/gpus-supported-by-vgpu.html

PCI passthrough
===============

* https://docs.openstack.org/nova/latest/admin/pci-passthrough.html
* https://docs.openstack.org/nova/latest/configuration/config.html#pci

* check IOMMU support

   * https://www.linux-kvm.org/page/How_to_assign_devices_with_VT-d_in_KVM

  .. code-block:: console

     $ dmesg | grep IOMMU
     [    0.000000] DMAR: IOMMU enabled
     [    0.207515] DMAR-IR: IOAPIC id 12 under DRHD base  0xc5ffc000 IOMMU 6
     [    0.207516] DMAR-IR: IOAPIC id 11 under DRHD base  0xb87fc000 IOMMU 5
     [    0.207518] DMAR-IR: IOAPIC id 10 under DRHD base  0xaaffc000 IOMMU 4
     [    0.207519] DMAR-IR: IOAPIC id 18 under DRHD base  0xfbffc000 IOMMU 3
     [    0.207520] DMAR-IR: IOAPIC id 17 under DRHD base  0xee7fc000 IOMMU 2
     [    0.207522] DMAR-IR: IOAPIC id 16 under DRHD base  0xe0ffc000 IOMMU 1
     [    0.207523] DMAR-IR: IOAPIC id 15 under DRHD base  0xd37fc000 IOMMU 0
     [    0.207525] DMAR-IR: IOAPIC id 8 under DRHD base  0x9d7fc000 IOMMU 7
     [    0.207526] DMAR-IR: IOAPIC id 9 under DRHD base  0x9d7fc000 IOMMU 7

* IOMMU PASS

  .. code-block:: console

     $ docker exec -it nova_libvirt virt-host-validate
     [...]
     QEMU: Checking for device assignment IOMMU support                         : PASS
     QEMU: Checking if IOMMU is enabled by kernel                               : PASS
     [...]

* IOMMU WARN

  .. code-block:: console

     $ docker exec -it nova_libvirt virt-host-validate
     [...]
     QEMU: Checking for device assignment IOMMU support                         : PASS
     QEMU: Checking if IOMMU is enabled by kernel                               : WARN (IOMMU appears to be disabled in kernel. Add intel_iommu=on to kernel cmdline arguments)
     [...]

* :ref:`enable-iommu`

* check if the nouveau kernel module is loaded

  .. code-block:: console

     $ lsmod | grep nouveau
     nouveau              1503232  0
     mxm_wmi                16384  1 nouveau
     video                  40960  1 nouveau
     ttm                    98304  2 ast,nouveau
     drm_kms_helper        155648  2 ast,nouveau
     drm                   364544  6 ast,ttm,drm_kms_helper,nouveau
     i2c_algo_bit           16384  3 ast,igb,nouveau
     wmi                    20480  2 mxm_wmi,nouveau

* disable nouveau in ``/etc/modprobe.d/blacklist-nvidia-nouveau.conf``

  .. code-block:: console

     blacklist nouveau
     blacklist snd_hda_intel # consumer gpu
     options nouveau modeset=0

* get vendor and product IDs

  .. code-block:: console

     $ lspci -nn

* enable PCI passthrough module in ``/etc/modprobe.d/vfio.conf``

  .. code-block:: console

     options vfio-pci ids=10de:1b38[,10de:1038]
     options vfio-pci disable_vga=1

* enable module after reboot in ``/etc/modules``

  .. code-block:: console

     ...
     vfio-pci

* rebuild the initramfs and reboot

  .. code-block:: console

     $ sudo update-initramfs -u
     $ sudo reboot

* check if ``vfio`` is used for ID

  .. code-block:: console

     $ lspci -nnk -s 84:00.0
     84:00.0 VGA compatible controller [0300]: NVIDIA Corporation GV102 [10de:1e07] (rev a1)
             Subsystem: Gigabyte Technology Co., Ltd Device [1458:37a9]
             Kernel driver in use: vfio-pci
             Kernel modules: nvidiafb, nouveau

* enable the ``PciPassthroughFilter`` scheduler in ``environments/kolla/files/overlays/nova/nova-scheduler.conf``

  .. code-block:: ini

     [filter_scheduler]
     enabled_filters = ..., PciPassthroughFilter

* specify PCI aliases for the devices in ``environments/kolla/files/overlays/nova/nova-api.conf`` and ``environments/kolla/files/overlays/nova/nova-compute.conf``

  .. code-block:: ini

     [pci]
     alias = { "vendor_id": "10de", "product_id":"1b38", "device_type":"type-PCI", "name":"nvidiap40" }
     alias = { "vendor_id": "10de", "product_id":"1adf", "device_type":"type-PCI", "name":"nvidiap40" }

Look into ``nova-compute.log`` on GPU-hypervisor for ``device_type``. Possible values are ``type-PCI``, ``type-PF`` or ``type-VF``.

* whitelist PCI devices in ``environments/kolla/files/overlays/nova/nova-compute.conf``

  .. code-block:: ini

     [pci]
     passthrough_whitelist = { "address": "0000:41:00.0" },
                             { "address": "0000:84:00.0" }

  or

  .. code-block:: ini

     [pci]
     passthrough_whitelist = { "vendor_id": "10de", "product_id": "1b38" },
                             { "vendor_id": "10de", "product_id": "1adf" }

.. note::

   In most environments not all compute nodes are equipped with a GPU. Store the compute node specific configurations in host specific overall files, e.g.
   in ``environments/kolla/files/overlays/nova/52-10.betacloud.xyz/nova.conf``.

   .. code-block:: ini

     [pci]
     alias = { "vendor_id": "10de", "product_id":"1b38", "device_type":"type-PCI", "name":"nvidiap40" }
     passthrough_whitelist = { "vendor_id": "10de", "product_id": "1b38" }

* set the ``pci_passthrough:alias"`` property on a flavor

  .. code-block:: console

     for one device
     $ openstack flavor set --property "pci_passthrough:alias"="nvidiap40:1" 1C-1G-1GB-10GB-GPU

     for more devices, the last digit is for the amount of devices
     $ openstack flavor set --property "pci_passthrough:alias"="nvidiap40:2" 1C-1G-1GB-10GB-GPU

Hidden KVM for Nvidia GPU
=========================

* set hidden feature in flavor

  .. code-block:: console

     $ openstack flavor set --property hide_hypervisor_id=true 1C-1G-1GB-10GB-GPU

Resource isolation
==================

* https://access.redhat.com/documentation/en-us/reference_architectures/2017/html/hyper-converged_red_hat_openstack_platform_10_and_red_hat_ceph_storage_2/tuning
* https://github.com/RHsyseng/hci/blob/master/scripts/nova_mem_cpu_calc.py

.. code-block:: console

   $ python nova_mem_cpu_calc.py HOST_MEMORY_GBYTE OSDS_PER_SERVER GUEST_AVG_MEMORY_GBYTE GUEST_AVG_CPU_UTIL
   $ python nova_mem_cpu_calc.py 256 56 6 8 0.1
   Inputs:
   - Total host RAM in GB: 256
   - Total host cores: 56
   - Ceph OSDs per host: 6
   - Average guest memory size in GB: 8
   - Average guest CPU utilization: 10%

   Results:
   - number of guests allowed based on memory = 28
   - number of guest vCPUs allowed = 500
   - nova.conf reserved_host_memory = 32000 MB
   - nova.conf cpu_allocation_ratio = 8.928571

Compare "guest vCPUs allowed" to "guests allowed based on memory" for actual guest count in ``environments/kolla/files/overlays/nova.conf``.

.. code-block:: ini

   [DEFAULT]
   reserved_host_cpus = 4
   reserved_host_memory_mb = 32768
   cpu_allocation_ratio = 9

Luks Encryption
===============

For a volume type witch uses Luks Encryption by keymanager it is required
to enable it in nova-compute config in the following section
``environments/kolla/files/overlays/nova/nova-compute.conf``.

.. code-block:: ini

   [[key_manager]
   backend = barbican
