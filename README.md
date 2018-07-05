# coreos-libvirt

### Purpose

I'm interested in [Docker](https://www.docker.com/) & want a more capable (better) Docker machine.  I've been using Docker with [CentOS 7](https://centos.org/) for a while.  While Docker works pretty well on CentOS 7, it's not the latest version and there are some limitations.

I'm not ready to move away from CentOS 7 completely, and figured a better way to accomplish my objectives would be to use [libvirt](https://libvirt.org/) on CentOS 7 to install [CoreOS](https://coreos.com/) virtual machines.  Then, run CoreOS & Docker in the virtual machines.  I also wanted to explore CoreOS some more, particularly [Ignition](https://coreos.com/ignition/) and the [container linux configuration transpiler](https://github.com/coreos/coreos-config-transpiler).

The physical machine I'm using is decent.  It's got 8 CPUs & 32GB RAM, plus a large hard drive.  It meets all of the requirements for qemu-kvm and is capable enough to do what I want.

### Objectives

* Automated, rapid setup & testing of 3+ node CoreOS (Container Linux) clusters, on qemu-kvm VMs, with etcd, etc.
* Bridged physical network, the Host all Guest VMs use external DHCP, no NATs.

### Procedure

1) CentOS 7 Host OS

   I'm using `CentOS Linux release 7.5.1804 (Core)`.  Getting that installed & setup is not the purpose of this document; I've had it setup for a while.  It might be worrth noting that I disabled biosdevname & use net.ifnames (e.g. eth0).  This was done (previously) by adding the following boot flags to linuxefi via grub.cfg `biosdevname=0 net.ifnames=0`.

2) Bridging eth0 & br0

    I use UUIDs for a reason.  STP was left on for a reason, too.  Others may want to set this up differently, but the objective is probably similar. The br0 interface uses an external, networked DHCP server for the Host OS.  I also want the qemu-kvm Guests to use the same external DHCP server (not provided by Host, i.e. _not_ dnsmasq).  The NetworkManager.service *_IS_* enabled (by default).

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

3) CentOS QEMU EV

    Doing this beforehand will avoid problems later.  The bottom line is that the stock CentOS 7 libvirt is outdated and doesn't support the `-fw_cfg` argument that the more modern CoreOS requires for Ignition to work properly.  The updated CentOS 7 QEMU EV packages do support that flag.
    ```
    sudo yum install epel-release centos-release-qemu-ev centos-release-virt-common
    ```

4) CentOS 7 QEMU EV libvirt packages

    I'm writing this after considerable trial and error.  This step may need to be revised.  The following list was captured after successful testing; libvirt was already installed when I began.
    ```
    sudo yum install libvirt libvirt-client libvirt-daemon libvirt-daemon-config-network libvirt-daemon-config-nwfilter libvirt-daemon-driver-interface libvirt-daemon-driver-lxc libvirt-daemon-driver-network libvirt-daemon-driver-nodedev libvirt-daemon-driver-nwfilter libvirt-daemon-driver-qemu libvirt-daemon-driver-secret libvirt-daemon-driver-storage libvirt-daemon-driver-storage-core libvirt-daemon-driver-storage-disk libvirt-daemon-driver-storage-gluster libvirt-daemon-driver-storage-iscsi libvirt-daemon-driver-storage-logical libvirt-daemon-driver-storage-mpath libvirt-daemon-driver-storage-rbd libvirt-daemon-driver-storage-scsi libvirt-daemon-kvm libvirt-devel libvirt-docs libvirt-gconfig libvirt-glib libvirt-gobject libvirt-java libvirt-java-devel libvirt-libs libvirt-python
    ```

5) CentOS 7 netfilters, firewalld, IP forwarding, etc.

    At one point, I ran into DCHP forwarding problems with libvirt.  This happened _only_ when running both Docker & libvirt on the Host (at the same time).  Docker adds its own stuff to the iptables FORWARD chain & that broke libvirt forwarding.  The solution was somewhat elusive.  Most referenced disabling netfilter for bridges via /etc/sysctl.conf.  FWIW, I did that like this ...
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
    * While the above works, _sort of_ & _most of the time_, it's not 100% reliable in my experience.  There was always an abnormally long delay obtaining the DHCP address inside the Guests.  Adding the following direct iptables rules via firewalld proved expedient & reliable for me (though I still left the above net.bridge... disables in /etc/sysctl.conf).
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

6) ignition.yml

    The following `build-coreos-libvirt` script does quite a few things.  Basically, it looks for a file called `ignition.yml`, transpiles it into `ignition-template.json`, and then pipes `ignition-template.json` through a sed filter to do some variable replacements.  Then it puts that into a dedicated ignition.json file fore each Guest VM, e.g. `vm/Node_Name/ignition.json`.

    Ultimately, this approach made it easier for me to modify the single `ignition.yml` and just keep re-running `build-coreos-libvirt` over and over to test multiple different variations of Ignition settings.  This is because _"**Ignition only runs once** and it does not handle variable substitution"_, which is [easy to miss in the documentation](https://coreos.com/ignition/docs/latest/what-is-ignition.html)
    * basic ignition.yml

    ```
    # https://github.com/coreos/container-linux-config-transpiler/blob/master/doc/configuration.md
    networkd:
      units:
        - name: 20-dhcp.network
          contents: |
            [Match]
            Name=e*

            [Network]
            DHCP=yes
    passwd:
      users:
        - name: core
          password_hash: $6$ct4dIwnHLjzCVb5W$CVtVgXNHkUuZRog6eXpORpfP4tvBVxY0jw4UYz5VfHZfJSzM4cP9Opl5D15KLdxbZCnaZqTF04lcWsf2nejcE1
          ssh_authorized_keys:
            - 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBVxxkTyq8DEBu5pfQzAzid4jIdDQhjDgVZGckzV/Q70'
            - 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBLPwh4f8WpCtb/EOpuz3U4ECTviSW7tBDQvWpndcD2q'
        - name: root
          password_hash: $6$ct4dIwnHLjzCVb5W$CVtVgXNHkUuZRog6eXpORpfP4tvBVxY0jw4UYz5VfHZfJSzM4cP9Opl5D15KLdxbZCnaZqTF04lcWsf2nejcE1
          ssh_authorized_keys:
            - 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBVxxkTyq8DEBu5pfQzAzid4jIdDQhjDgVZGckzV/Q70'
            - 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBLPwh4f8WpCtb/EOpuz3U4ECTviSW7tBDQvWpndcD2q'
    storage:
      files:
        - filesystem: "root"
          group:
          path: "/etc/hostname"
          user:
          mode: 0420
          contents:
            inline: "Node_Name"
    ```

7) build-coreos-libvirt

    [This is my script](https://github.com/josephtingiris/coreos-libvirt/blob/master/build-coreos-libvirt).  It could be better, but meets the objectives.  I really want to play with Ignition more & just need a way to rapidly tear down & rebuild qemu-kvm VMs appropriately.
    ```
    ./build-coreos-libvirt <virtual machine name prefix> <# of nodes> [ram] [vcpu]
    ```

8) Etcd

    At this point, provisioning the VM(s), booting, & remotely logging into CoreOS should work just fine.

    Configuring an actual cluster requires a 'discovery' service, and some decisions up front.  Since I'm all about DHCP & dynamic cloud services, setting up a static cluster is not an option.  The dynamic options are to use etcd or DNS for discovery.  [Regardless, all of the cluster discovery options are outlined, here.](https://coreos.com/etcd/docs/latest/v2/clustering.html) along with some important [caveats](https://coreos.com/etcd/docs/latest/v2/runtime-reconf-design.html)

    Time to Once I'd gotten the quirks worked out of qemu, bridging, & dhcp.  I ran into this bug:
    * [Couldn't find 'coreos.oem.id' flag in cmdline file (/proc/cmdline)](https://github.com/coreos/bugs/issues/2303)

### References

* [https://coreos.com/os/docs/latest/booting-with-qemu.html](https://coreos.com/os/docs/latest/booting-with-qemu.html)
* [https://coreos.com/os/docs/latest/booting-with-libvirt.html](https://coreos.com/os/docs/latest/booting-with-libvirt.html)

* To manually get & use config transpiler ...
    ```
    curl -L -O https://github.com/coreos/coreos-config-transpiler/releases/download/v0.9.0/ct-v0.9.0-x86_64-unknown-linux-gnu
    ln -s ct-v0.9.0-x86_64-unknown-linux-gnu ct
    ct -in-file ignition.yml -pretty > ignition-template.json && cat ignition-template.json

    ```

* To manually generate an SHA512 hash for ignition.yml ...
    ```
    python -c 'import crypt,getpass; print(crypt.crypt(getpass.getpass(), crypt.mksalt(crypt.METHOD_SHA512)))'
    ```

* etcd ...

    * [how to get a discovery key is here](https://coreos.com/etcd/docs/latest/v2/clustering.html)
    * [curl https://discovery.etcd.io/new?size=3](https://discovery.etcd.io/new?size=3)

