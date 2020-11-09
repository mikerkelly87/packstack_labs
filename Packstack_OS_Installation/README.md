# OS Installation

This installation can be done either in a VM or on a physical machine.
The minimum requirements straight from the [RDO Project](https://www.rdoproject.org/install/packstack/) is the following:
```
Machine with at least 16GB RAM
Processors with hardware virtualization extensions
At least one network adapter
```
You can probably get away with the following:
```
2 vCPU
12GB of RAM
100GB SSD/HDD
1 NIC
```
If you're going to be running this in a VM there are two things to be aware of
```
1) You need to make sure the VM supports nested virtualization, without this the Openstack instances will not boot.
You will need to refer to the documentation for your specific hypervisor to learn how to enable this.

2) The network interface on your VM needs to be a bridged interface.
Again you will need to refer to the documentation for your specific hypervisor to learn how to configure this.
The end result will be the VM existing directly on the same CIDR as your home LAN.

3) Some hypervisors, such as VirtualBox, will require you to enable Promiscuous mode on the VM's network adapter.
```

If you need steps to installing a CentOS 8 minimal installation there are steps provided here:
[OS Installation](os_install.md)

[<-- Back to Main](../README.md)
