VM (on Ubuntu)
==============

(There are detailed instructions available below, the overview and
configuration sections provide background information).

Overview:
* Setup a VM that is your bootstrap node
* Setup N VMs to pretend to be your cluster
* Go to town testing deployments on them.
* For troubleshooting see [troubleshooting.md](troubleshooting.md)

Configuration
-------------

The bootstrap instance expects to run with its eth0 connected to the outside
world, via whatever IP range you choose to setup. You can run NAT, or not, as
you choose. This is how we connect to it to run scripts etc - though you can
equally log in on its console if you like.

As we have not yet taught quantum how to deploy VLANs to bare metal instances,
we're using flat networking, with a single broadcast domain which all the bare
metal machines are connected to.

According, the eth1 of your bootstrap instance should be connected to your bare
metal cloud LAN. The bootstrap VM uses the rfc5735 TEST-NET-1 range -
192.0.2.0/24 for bringing up nodes, and does its own DHCP etc, so do not
connect it to a network shared with other DHCP servers or the like. The
instructions in this document create a bridge device ('br99') on your
machine to emulate this with virtual machine 'bare metal' nodes.


  NOTE: We recommend using an apt/HTTP proxy and setting the http_proxy
        environment variable accordingly.

  NOTE: Building images will be extremely slow on Ubuntu 12.04 (precise). This
        is due to nbd-qemu lacking writeback caching. Using 12.10 will be
        significantly faster.

  NOTE: The CPU architecture specified in several places must be consistent.
        This document's examples use 32-bit arch for the reduced memory footprint.
        If you are running on real hardware, or want to test with 64-bit arch,
        replace i386 => amd64 and i686 => x86_64 in all the commands below.
        Also, you need to edit incubator/localrc and change BM_CPU_ARCH accordingly.

Detailed instructions
---------------------

* Before you start, check to see that your machine supports hardware
  virtualization, otherwise KVM is going to get very grumpy.

* Also check ssh server is running on the host machine and port 22 is open. 
  VirtPowerManager will boot vms by ssh'ing into the host machine and 
  issuing libvirt/virsh commands.

* Choose a base location to put all of the source code.

        mkdir ~/tripleo
        export TRIPLEO_ROOT=~/tripleo
        cd $TRIPLEO_ROOT

* git clone this repository to your local machine.

        git clone https://github.com/tripleo/incubator.git

* git clone bm_poseur to your local machine.

        git clone https://github.com/tripleo/bm_poseur.git

* git clone diskimage-builder and the tripleo elements likewise.

        git clone https://github.com/stackforge/diskimage-builder.git
        git clone https://github.com/stackforge/tripleo-image-elements.git

* Ensure dependencies are installed and required virsh configuration is performed:

        cd $TRIPLEO_ROOT/incubator
        scripts/install-dependencies

* Configure a network for your test environment.
  (This alters your /etc/network/interfaces file and adds an exclusion for
  dnsmasq so it doesn't listen on your test network.)

        cd $TRIPLEO_ROOT/bm_poseur/
        sudo ./bm_poseur --bridge-ip=none create-bridge

* Activate these changes (alternatively, restart):

        sudo service libvirt-bin restart

* Create your machine image. This is the image that baremetal nova
  will install on each node. You can also download a pre-built image,
  or experiment with different combinations of elements.

        cd $TRIPLEO_ROOT/diskimage-builder/
        bin/disk-image-create -u base -a i386 -o $TRIPLEO_ROOT/incubator/base

* Create and start your bootstrap VM. This script invokes diskimage-builder
  with suitable paths and options to create and start a VM that contains an
  all-in-one OpenStack cloud with the baremetal driver enabled, and preconfigures
  it for a development environment.

        cd $TRIPLEO_ROOT/tripleo-image-elements/elements/boot-stack
        sed -i "s/\"user\": \"stack\",/\"user\": \"`whoami`\",/" config.json

        cd $TRIPLEO_ROOT/incubator/
        scripts/boot-elements boot-stack -o bootstrap

  Your SSH pub key has been copied to the resulting 'bootstrap' VM's root user.
  It has been started by the boot-elements script, and can be logged into at this point.

* Get the IP of your 'bootstrap' VM

        BOOTSTRAP_IP=`scripts/get-vm-ip bootstrap`

  (If you downloaded a pre-built bootstrap image, or chose not to start it by
  specifying the -n option, you will need to manually start it and customize it.
  See footnote [1].)

* Create some 'baremetal' node(s) out of KVM virtual machines.
  Nova will PXE boot these VMs as though they were physical hardware.
  You can use bm_poseur to automate this, or if you want to create
  the VMs yourself, see footnote [2] for details on their requirements.

        sudo $TRIPLEO_ROOT/bm_poseur/bm_poseur --vms 1 --arch i686 create-vm

* Get the list of MAC addresses for all the VMs you have created.
  If you used bm_poseur to create the bare metal nodes, you can run this
  on your laptop to get the MACs:

        MAC=`$TRIPLEO_ROOT/bm_poseur/bm_poseur get-macs`

  If you are testing on real hardware, see footnote [3].

* Copy the openstack credentials out of the bootstrap VM, and add the IP:

        scp root@$BOOTSTRAP_IP:stackrc ~/stackrc
        sed -i "s/localhost/$BOOTSTRAP_IP/" ~/stackrc
        source ~/stackrc

__(Note: all of the following commands should be run on your host machine, not inside the bootstrap VM)__
__(Note: if you have set http_proxy or https_proxy to a network host, these need to be unset before proceeding)__

* Add your key to nova:

        nova keypair-add --pub-key ~/.ssh/id_rsa.pub default

* Inform Nova on the bootstrap node of these resources by running this:

        nova baremetal-node-create ubuntu 1 512 10 $MAC

  If you have multiple VMs created by bm_poseur, you can simplify this process
  by running this script.

        for MAC in $($TRIPLEO_ROOT/bm_poseur/bm_poseur get-macs); do
            nova baremetal-node-create ubuntu 1 512 10 $MAC
        done

  (This assumes the default flavor of CPU:1 RAM:512 DISK:10. Change values if needed.)

* Wait for the following to show up in the nova-compute log on the bootstrap node

        ssh root@$BOOTSTRAP_IP "tail -f /var/log/upstart/nova-compute.log"

        2013-01-08 16:43:13 AUDIT nova.compute.resource_tracker [-] Auditing locally available compute resources
        2013-01-08 16:43:13 DEBUG nova.compute.resource_tracker [-] Hypervisor: free ram (MB): 512 from (pid=24853) _report_hypervisor_resource_view /opt/stack/nova/nova/compute/resource_tracker.py:327
        2013-01-08 16:43:13 DEBUG nova.compute.resource_tracker [-] Hypervisor: free disk (GB): 0 from (pid=24853) _report_hypervisor_resource_view /opt/stack/nova/nova/compute/resource_tracker.py:328
        2013-01-08 16:43:13 DEBUG nova.compute.resource_tracker [-] Hypervisor: free VCPUs: 1 from (pid=24853) _report_hypervisor_resource_view /opt/stack/nova/nova/compute/resource_tracker.py:333
        2013-01-08 16:43:13 AUDIT nova.compute.resource_tracker [-] Free ram (MB): 0
        2013-01-08 16:43:13 AUDIT nova.compute.resource_tracker [-] Free disk (GB): 0
        2013-01-08 16:43:13 AUDIT nova.compute.resource_tracker [-] Free VCPUS: 1

* Load the base image into Glance:

        cd $TRIPLEO_ROOT/incubator/
        scripts/load-image base.qcow2

* Allow the VirtualPowerManager to ssh into your host machine to power on vms:

        ssh root@$BOOTSTRAP_IP "cat /opt/stack/boot-stack/virtual-power-key.pub" >> ~/.ssh/authorized_keys

* Start the process of provisioning a baremetal node:

        nova boot --flavor 256 --image base --key_name default bmtest

  You can watch its console to observe the PXE boot/deploy process.
  After the deploy is complete, it will reboot into the base image.


  The End!



Footnotes
=========

* [1] Customize a downloaded bootstrap image.

  If you downloaded your bootstrap VM's image, you may need to configure it.
  Setup a network proxy, if you have one (e.g. 192.168.2.1 port 8080)

        echo << EOF >> ~/.profile
        export no_proxy=192.0.2.1
        export http_proxy=http://192.168.2.1:8080/
        EOF

  Add an ~/.ssh/authorized_keys file. The image rejects password authentication
  for security, so you will need to ssh out from the VM console. Even if you
  don't copy your authorized_keys in, you will still need to ensure that
  /home/stack/.ssh/authorized_keys on your bootstrap node has some kind of
  public SSH key in it, or the openstack configuration scripts will error.

* [2] Requirements for the "baremetal node" VMs

  If you don't use bm_poseur, but want to create your own VMs, here are some
  suggestions for what they should look like.
   - each VM should have two NICs
   - both NICs should be on br99
   - record the MAC addresses for each NIC
   - give each VM no less than 2GB of disk, and ideally give them
     more than BM_FLAVOR_ROOT_DISK, which defaults to 2GB
   - 512MB RAM is probably enough
   - if using KVM, specify that you will install the virtual machine via PXE.
     This will avoid KVM prompting for a disk image or installation media.

* [3] Notes for physical hardware environments

  If you are running a different environment, e.g. real hardware, you will
  need to edit the incubator environment as needed within the bootstrap
  node prior to running incubator/scripts/demo.

  See localrc and scripts/defaults in the incubator tree on your bootstrap node.
  Also see devstack/lib/baremetal for a full list of options that can
  inform Nova of the environment.

