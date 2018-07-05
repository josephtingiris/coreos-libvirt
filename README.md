# coreos-libvirt

### Purpose

I'm interested in [Docker](https://www.docker.com/) & want a more capable (better) Docker machine.  I've been using Docker with [CentOS 7](https://centos.org/) for a while.  While Docker works pretty well on CentOS 7, it's not the latest version and there are some limitations.

I'm not ready to move away from CentOS 7 completely, and figured a better way to accomplish my objectives would be to use [libvirt](https://libvirt.org/) on CentOS 7 to install [CoreOS](https://coreos.com/) virtual machines.  Then, run CoreOS & Docker in the virtual machines.  I also wanted to explore CoreOS some more, particularly [Ignition](https://coreos.com/ignition/) and the [container linux configuration transpiler](https://github.com/coreos/coreos-config-transpiler).

The physical machine I'm using is decent.  It's got 8 CPUs & 32GB RAM, plus a decent sized hard drive.  It meets all of the requirements for qemu-kvm and is capable enough to do what I want.

### Objectives

* Automated, rapid setup & testing of 3+ node CoreOS (Container Linux) cluster, on qemu-kvm VMs, with etcd.
* Bridged network, all VMs use external DHCP, no NATs

1) CentOS 7 Host OS

   I'm using `CentOS Linux release 7.5.1804 (Core)` I also disabled biosdevname & use legacy net.ifnames (e.g. eth0).  This was done by adding the following boot flags to linuxefi via grub.cfg `biosdevname=0 net.ifnames=0`

1) Bridging eth0 & br0

    I use UUIDs for a reason.  STP was left on for a reason, too.  Others may or may want to set this up differently, but the objective is probably similar. The br0 interface uses an external DHCP server for the Host OS.  I also want the qemu-kvm Guests to use the same external DHCP server (not dnsmasq on the Host).

    * /etc/sysconfig/network-scripts/ifcfg-eth0
    ```
    BRIDGE=c0ca800d-482a-4ca4-999b-7884d832410d
    BOOTPROTO=none
    DEVICE=eth0
    NAME=eth0
    TYPE=Ethernet
    UUID=54b70252-2d3f-4b93-9be0-9f62dade632c
    ONBOOT=yes
    ```
    * /etc/sysconfig/network-scripts/ifcfg-br0
    ```
    DEVICE=br0
    STP=yes
    TYPE=Bridge
    BOOTPROTO=dhcp
    DEFROUTE=yes
    IPV4_FAILURE_FATAL=no
    IPV6INIT=yes
    IPV6_AUTOCONF=yes
    IPV6_DEFROUTE=yes
    IPV6_FAILURE_FATAL=no
    NAME=br0
    UUID=c0ca800d-482a-4ca4-999b-7884d832410d
    ONBOOT=yes
    BRIDGING_OPTS=priority=32768
    DOMAIN="mydomain.com myotherdomain.com"
    PROXY_METHOD=none
    BROWSER_ONLY=no
    ```

2) CentOS QEMU EV

    Doing this beforehand will avoid problems later.  The bottom line is that the default CentOS 7 libvirt is outdated and doesn't support the `-fw_cfg` argument that more modern CoreOS requires for Ignition to work properly.  The updated CentOS QEMU EV packages do.
    ```
    sudo yum install epel-release centos-release-qemu-ev centos-release-virt-common
    ```

3) CentOS 7 QEMU EV libvirt packages

    The following list was captured after the fact.  I'm writing this after considerable trial and error.  This step may need to be revised.
    ```
    sudo yum install libvirt libvirt-client libvirt-daemon libvirt-daemon-config-network libvirt-daemon-config-nwfilter libvirt-daemon-driver-interface libvirt-daemon-driver-lxc libvirt-daemon-driver-network libvirt-daemon-driver-nodedev libvirt-daemon-driver-nwfilter libvirt-daemon-driver-qemu libvirt-daemon-driver-secret libvirt-daemon-driver-storage libvirt-daemon-driver-storage-core libvirt-daemon-driver-storage-disk libvirt-daemon-driver-storage-gluster libvirt-daemon-driver-storage-iscsi libvirt-daemon-driver-storage-logical libvirt-daemon-driver-storage-mpath libvirt-daemon-driver-storage-rbd libvirt-daemon-driver-storage-scsi libvirt-daemon-kvm libvirt-devel libvirt-docs libvirt-gconfig libvirt-glib libvirt-gobject libvirt-java libvirt-java-devel libvirt-libs libvirt-python
    ```

4) CentOS 7 netfilters, firewalld, IP forwarding, etc.

    I ran into forwarding problems with libvirt properly forwarding DHCP.  This happened _only_ when running both Docker & libvirt on the Host (at the same time).  It was a bit frustrating and the solution was somewhat elusive.  Most referenced disabling netfilter for bridges via /etc/sysctl.conf.  FWIW, here's mine ...
    * /etc/sysctl.conf
    ```
    # System default settings live in /usr/lib/sysctl.d/00-system.conf.
    # To override those settings, enter new settings here, or in an /etc/sysctl.d/<name>.conf file
    #
    # For more information, see sysctl.conf(5) and sysctl.d(5).
    net.ipv4.ip_forward=1
    net.ipv6.conf.all.forwarding=1
    net.bridge.bridge-nf-call-iptables = 0
    net.bridge.bridge-nf-call-ip6tables = 0
    net.bridge.bridge-nf-call-arptables = 0
    net.bridge.bridge-nf-filter-vlan-tagged = 0
    ```
    * While the above works, _sort of_ & _most of the time_, it's not 100% reliable in my experience.  Adding the following direct rules via firewalld have proven reliable for me ....
        ```
        firewall-cmd --permanent --direct --passthrough ipv4 -I FORWARD -i br0 -j ACCEPT
        firewall-cmd --permanent --direct --passthrough ipv4 -I FORWARD -o br0 -j ACCEPT
        firewall-cmd --permanent --direct --passthrough ipv6 -I FORWARD -i br0 -j ACCEPT
        firewall-cmd --permanent --direct --passthrough ipv6 -I FORWARD -o br0 -j ACCEPT
        systemctl restart firewalld
        ```
        See:
        * [libvirt Networking](https://wiki.libvirt.org/page/Networking)
        * [related bug](https://bugzilla.redhat.com/show_bug.cgi?id=512206)

5) build-coreos-libvirt

    This is my script.  It meets the objectives.
    ```
    ./build-coreos-libvirt <virtual machine name prefix> <# of nodes> [ram] [vcpu]
    ```

### References

* [https://coreos.com/os/docs/latest/booting-with-qemu.html](https://coreos.com/os/docs/latest/booting-with-qemu.html)
* [https://coreos.com/os/docs/latest/booting-with-libvirt.html](https://coreos.com/os/docs/latest/booting-with-libvirt.html)

* To get & use config transpiler ...
    ```
    curl -L -O https://github.com/coreos/coreos-config-transpiler/releases/download/v0.9.0/ct-v0.9.0-x86_64-unknown-linux-gnu
    ln -s ct-v0.9.0-x86_64-unknown-linux-gnu ct
    ct -in-file ignition.yml -pretty > ignition-template.json && cat ignition-template.json

    ```

* To generate an SHA512 has for ignition.yml ...
    ```
    python -c 'import crypt,getpass; print(crypt.crypt(getpass.getpass(), crypt.mksalt(crypt.METHOD_SHA512)))'
    ```

* etcd ...

    * [how to get a discovery key is here](https://coreos.com/etcd/docs/latest/v2/clustering.html)
    * [curl https://discovery.etcd.io/new?size=3](https://discovery.etcd.io/new?size=3)

