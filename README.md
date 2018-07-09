# coreos-libvirt

### Purpose

I'm interested in [Docker](https://www.docker.com/) & want more capable (better) Docker machines.  For the past couple years, I've been using Docker with [CentOS 7](https://centos.org/) (primarily).  While Docker works pretty well on CentOS 7, it's not the latest version and there are some limitations.

I'm not ready to move away from CentOS/RHEL completely, and figured a better way to accomplish my objectives would be to use [libvirt](https://libvirt.org/) on CentOS 7 to install [CoreOS](https://coreos.com/) virtual machines.  Then, run CoreOS & Docker in the virtual machines.  I also wanted to explore CoreOS some more, particularly its [Ignition](https://coreos.com/ignition/) and the [container linux configuration transpiler](https://github.com/coreos/coreos-config-transpiler).

The physical machines I'm using are decent.  At minimum, I've got 8 CPUs & 32GB RAM, plus a large hard drive.  It meets all of the requirements for qemu-kvm and is capable enough to do what I want.

### Objectives

* Automate the rapid set up, tear down, & testing of 3+ node CoreOS (Container Linux) sparse clusters, on qemu-kvm VMs, with etcd, docker, swarm, kubernetes, etc.
* Use bridged interfaces for networking; the Host & all Guest VMs should use external DHCP, no NATs.

### Procedure

1) CentOS 7 Host OS

   I'm testing on _CentOS Linux release 7.5.1804 (Core)_ and I've had it setup for a while (since 7.1).  Getting that installed & setup is not the purpose of this document.  Though it might be worth noting that I disabled biosdevname & use net.ifnames (e.g. eth0).  This was done (previously) by adding the following boot flags to the grub.cfg linuxefi lines ... `biosdevname=0 net.ifnames=0`

2) Bridging eth0 & br0

    I use UUIDs & STP for a reason.  Others machines may be set up differently, but the objective is probably similar. The br0 interface uses an external, networked DHCP server for the Host OS.  I also want the qemu-kvm Guests to use the same external DHCP server (not provided by Host, i.e. _not_ dnsmasq) and have their own (dynamic) IP v4 & v6 addresses.  The NetworkManager.service *_IS_* enabled (by default) and running on the Host.

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

    Doing this beforehand will avoid problems later.  The bottom line is that the stock CentOS 7 libvirt is outdated and doesn't support the `-fw_cfg` argument that the more modern CoreOS requires for Ignition to work properly.  The updated CentOS 7 QEMU EV packages are newer and do support that flag.
    ```
    sudo yum install epel-release centos-release-qemu-ev centos-release-virt-common
    ```

4) CentOS 7 QEMU EV libvirt packages

    I'm writing this after trial and error.  I'll likily come back and this step may need to be revised.  The following list was captured after successful testing; libvirt was already installed when I began.
    ```
    sudo yum install libvirt libvirt-client libvirt-daemon libvirt-daemon-config-network libvirt-daemon-config-nwfilter libvirt-daemon-driver-interface libvirt-daemon-driver-lxc libvirt-daemon-driver-network libvirt-daemon-driver-nodedev libvirt-daemon-driver-nwfilter libvirt-daemon-driver-qemu libvirt-daemon-driver-secret libvirt-daemon-driver-storage libvirt-daemon-driver-storage-core libvirt-daemon-driver-storage-disk libvirt-daemon-driver-storage-gluster libvirt-daemon-driver-storage-iscsi libvirt-daemon-driver-storage-logical libvirt-daemon-driver-storage-mpath libvirt-daemon-driver-storage-rbd libvirt-daemon-driver-storage-scsi libvirt-daemon-kvm libvirt-devel libvirt-docs libvirt-gconfig libvirt-glib libvirt-gobject libvirt-java libvirt-java-devel libvirt-libs libvirt-python
    ```

5) CentOS 7 netfilters, firewalld, IP forwarding, etc.

    I ran into DCHP problems & IP forwarding with libvirt.  This happened most often when running both Docker & libvirt on the Host machine at the same time.  When containers are started, Docker dynamically adds its own chains to the iptables & that would break libvirt IP forwarding (which also dynamically updates iptables).  The solution was somewhat elusive and not as straightforward as the [libvirt Networking](https://wiki.libvirt.org/page/Networking) documentation alluded to.  It referenced disabling netfilter for bridges via /etc/sysctl.conf.  FWIW, I did that like this ...

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

    * While the above worked, _sort of_ & _most of the time_, [it wasn't 100% reliable](https://bugzilla.redhat.com/show_bug.cgi?id=512206).  I'm constantly running different containers, my iptables change frequently, and there was often an abnormally long delay obtaining the DHCP address inside the Guests.  Adding the following direct iptables rules to the Host machine, via firewalld, proved expedient & reliable for me (though I still left the net.bridge-nf* disabled in /etc/sysctl.conf).
        ```
        firewall-cmd --permanent --direct --passthrough ipv4 -I FORWARD -i br0 -j ACCEPT
        firewall-cmd --permanent --direct --passthrough ipv4 -I FORWARD -o br0 -j ACCEPT
        firewall-cmd --permanent --direct --passthrough ipv6 -I FORWARD -i br0 -j ACCEPT
        firewall-cmd --permanent --direct --passthrough ipv6 -I FORWARD -o br0 -j ACCEPT
        systemctl restart firewalld
        ```

    * Finally, during testing, I initially ran out of DHCP leases on my subnet for two reasons.  This is mostly obsfucated & automated in the steps below. Here are some of the details.

        1) libvirt generates a different mac adderess every time [virt-install](https://github.com/virt-manager/virt-manager/blob/master/man/virt-install.pod) is run.  In this situation, it's expected that an DHCP server would assign a new lease to the 'new' machine.  I've worked around that before by coming up with a 'consistent' mac address algorithm in my test scripts.  Basically, the node name is hashed & then cut so that the resulting qemu-kvm mac address is always the same (for the same node name).  This got me part of the way ...

        2) ... until I realized [systemd-networkd](http://man7.org/linux/man-pages/man5/systemd.network.5.html) works differently than most DHCP clients.   Regardless of a consistent mac address, every time a new machine was setup the DHCP discover _still_ got a new IP address offered and the [DHCP IP changes on every boot](https://github.com/coreos/bugs/issues/1432).  Apparently systemd-networkd generates an unique DHCPv4 client identifier and uses it by default for DHCP discovery, despite the fact that the mac address _is_ consistent.  Brilliant.  I'm not alone in disagreeing with that approach, but I was able to fix it by adding `ClientIdentifier=mac` to the networkd: section of my `ignition.yml` files.

6) ignition.yml

    [When Ignition is executed](https://coreos.com/ignition/docs/latest/what-is-ignition.html#when-is-ignition-executed), it provisions the (virtual) machine.  The so-called `ignition.yml` file contains all of the initial [configuration transpiler directives](https://github.com/coreos/container-linux-config-transpiler/blob/master/doc/configuration.md).  Once it's run through [ct](https://github.com/coreos/coreos-config-transpiler/releases/download/) then it gets validated & converted to JSON.

    The following `build-coreos-libvirt` looks for a file named `ignition.yml`.  Basically, it runs `virt-install` to create a qemu-kvm virtual machine, looks for a file called `ignition.yml` in the same directory, transpiles that into `ignition-template.json`, and then pipes `ignition-template.json` through a sed filter to do some variable replacements.  Then it redirects that ouput into a dedicated `ignition.json` file fore each Guest VM that's built, e.g. `vm/Node_Name/ignition.json`.

    * ignition.yml (basic dhcp, hostname, & users)
    ```
    # https://github.com/coreos/container-linux-config-transpiler/blob/master/doc/configuration.md
    networkd:
      units:
        - name: 10-dhcp.network
          contents: |
            # http://man7.org/linux/man-pages/man5/systemd.network.5.html
            [Match]
            Name=eth*

            [Network]
            DHCP=yes

            [DHCP]
            ClientIdentifier=mac
            UseDomains=true
            UseNTP=false
    passwd:
      users:
        - name: core
          password_hash: $6$N5fRCDvwrWLn7gZO$ht2G4/pCy7yPbrO6hvWdTn.ATErEKfI3f4NIc3N5Wx9oGJJGP7xCys5qi6geB1QJRB/uBQm3su7L6b6QREsCu.
          ssh_authorized_keys:
            - 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKp2i3yIUEejbxXRxuLM6nnkE7EZVdabmULxXYrqesr5'
            - 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILCkJFCCHiOf9Wi8ztbKB2liC0+mrUKAIEi38u7gM39R'
        - name: root
          password_hash: $6$DEjx9uG1rgRNLSzl$YdMxRZFlhQ4Ya086zYwkDq.nzW/.hg07uxR6fnBTDOUdG0Yb8YWw.8HabhQvNRe5rw.5rheJ7BO9f5OkuRXBo0
          ssh_authorized_keys:
            - 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKp2i3yIUEejbxXRxuLM6nnkE7EZVdabmULxXYrqesr5'
            - 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILCkJFCCHiOf9Wi8ztbKB2liC0+mrUKAIEi38u7gM39R'
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

    [This is my script](https://github.com/josephtingiris/coreos-libvirt/blob/master/build-coreos-libvirt).  It could be better, but meets the objectives.  I didn't wnat to spend too much time, here, and really want to play with Ignition more & just need a way to rapidly tear down & rebuild qemu-kvm VMs appropriately.
    ```
    ./build-coreos-libvirt <virtual machine (node) name prefix> <# of nodes> [ram] [vcpu]
    ```

    Ultimately, this approach makes it easier (and faster) for me to modify the single `ignition.yml` input file and just keep re-running `build-coreos-libvirt` to test multiple different variations of Ignition settings.  This is because _"**Ignition only runs once** and it does not handle variable substitution"_, which I found [easy to miss in the documentation](https://coreos.com/ignition/docs/latest/what-is-ignition.html).  I understand the logic, surely.  Getting _all_ of the kinks worked out of a system _before_ it goes into daily use is important.


8) coreos-metadata

    If using the above `iginition.yml` and `build-coreos-libvirt` then at this point rapidly provisioning the VM(s), booting them, & remotely logging into CoreOS should work just fine, providing the `ignition.yml` has valid `ssh_authorized_keys` and/or a valid `password_hash`.

    Once I'd gotten the quirks worked out of qemu-kvm, bridging, dhcp, etc.  My first attempts at getting etcd working were met with the message '[Couldn't find 'coreos.oem.id' flag in cmdline file (/proc/cmdline)](https://github.com/coreos/bugs/issues/2303)', which is a a systemd dependency of `etcd-member.service`.

    Since I'm using DHCP, I need to use the IP addresses that are assign.  But, I wont always know that.  Aha, metadata!  That lead to the understanding & creation of a [custom metadata provider](https://github.com/coreos/container-linux-config-transpiler/blob/v0.8.0/doc/dynamic-data.md#custom-metadata-providers).

    I called my provider [coreos-metadata-qemu](https://github.com/josephtingiris/coreos-libvirt/blob/master/coreos-metadata-qemu), put it in the same directory as `build-coreos-libvirt`, and added it to my `ignition.yml` with bits like this.  It's important to call it _After_ `network-online.target` so that it gets the DHCP assigned addresses and puts them in **/run/metadata/coreos** so that systemd can use them as environment variables.  There are other ways to accomplish this, but something like this works well enough.
    ```
    storage:
      files:
        - filesystem: "root"
          group:
          path: "/etc/hostname"
          user:
          mode: 0420
          contents:
            inline: "Node_Name"
        - filesystem: "root"
          group:
          path: "/opt/custom/bin/coreos-metadata-qemu"
          user:
          mode: 0755
          contents:
            local: coreos-metadata-qemu
    systemd:
      units:
        - name: "coreos-metadata.service"
          dropins:
            - name: "coreos-metadata-qemu.conf"
              contents: |
                # https://github.com/coreos/container-linux-config-transpiler/blob/v0.8.0/doc/dynamic-data.md#custom-metadata-providers
                [Unit]
                After=network-online.target
                [Service]
                ExecStart=
                ExecStart=/opt/custom/bin/coreos-metadata-qemu
    ```

9) etcd

    Configuring a CoreOS cluster requires a 'discovery' service and some decisions up front.  Since I'm using DHCP, setting up a static cluster is not an option.  Unfortunately, the CoreOS documentation for DHCP is rather second-hand.

    The available dynamic options are to use either etcd or DNS for discovery.  [Regardless, all of the cluster discovery options are outlined, here.](https://coreos.com/etcd/docs/latest/v2/clustering.html) along with some important [caveats](https://coreos.com/etcd/docs/latest/v2/runtime-reconf-design.html)

    NOTE: `build-coreos-libvirt` automatically replaces _Etcd_Discovery_URL_, _Node_Name_, & _Node_Prefix_.
    ```
    # https://github.com/coreos/container-linux-config-transpiler/blob/master/doc/configuration.md
    etcd:
      name:                        "{HOSTNAME}"
      listen_client_urls:          "http://0.0.0.0:2379,http://0.0.0.0:4001"
      advertise_client_urls:       "http://{PUBLIC_IPV4}:2379,http://{PUBLIC_IPV4}:4001"
      listen_peer_urls:            "http://0.0.0.0:2380,http://0.0.0.0:7001"
      initial_advertise_peer_urls: "http://{PUBLIC_IPV4}:2380,http://{PUBLIC_IPV4}:7001"
      discovery:                   "Etcd_Discovery_URL"
    networkd:
      units:
        - name: 10-dhcp.network
          contents: |
            # http://man7.org/linux/man-pages/man5/systemd.network.5.html
            [Match]
            Name=eth*

            [Network]
            DHCP=yes

            [DHCP]
            ClientIdentifier=mac
            UseDomains=true
            UseNTP=false
    passwd:
      users:
        - name: core
          password_hash: $6$N5fRCDvwrWLn7gZO$ht2G4/pCy7yPbrO6hvWdTn.ATErEKfI3f4NIc3N5Wx9oGJJGP7xCys5qi6geB1QJRB/uBQm3su7L6b6QREsCu.
          ssh_authorized_keys:
            - 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKp2i3yIUEejbxXRxuLM6nnkE7EZVdabmULxXYrqesr5'
            - 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILCkJFCCHiOf9Wi8ztbKB2liC0+mrUKAIEi38u7gM39R'
        - name: root
          password_hash: $6$DEjx9uG1rgRNLSzl$YdMxRZFlhQ4Ya086zYwkDq.nzW/.hg07uxR6fnBTDOUdG0Yb8YWw.8HabhQvNRe5rw.5rheJ7BO9f5OkuRXBo0
          ssh_authorized_keys:
            - 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKp2i3yIUEejbxXRxuLM6nnkE7EZVdabmULxXYrqesr5'
            - 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILCkJFCCHiOf9Wi8ztbKB2liC0+mrUKAIEi38u7gM39R'
    storage:
      files:
        - filesystem: "root"
          group:
          path: "/etc/hostname"
          user:
          mode: 0420
          contents:
            inline: "Node_Name"
        - filesystem: "root"
          group:
          path: "/opt/custom/bin/coreos-metadata-qemu"
          user:
          mode: 0755
          contents:
            local: coreos-metadata-qemu
    systemd:
      units:
        - name: "coreos-metadata.service"
          dropins:
            - name: "coreos-metadata-qemu.conf"
              contents: |
                # https://github.com/coreos/container-linux-config-transpiler/blob/v0.8.0/doc/dynamic-data.md#custom-metadata-providers
                [Unit]
                After=network-online.target
                [Service]
                ExecStart=
                ExecStart=/opt/custom/bin/coreos-metadata-qemu
        - name: etcd-member.service
          enabled: true
          dropins:
            - name: Node_Name.conf
              contents: |
                [Unit]
                After=coreos-metadata.service
                [Service]
                Environment="ETCD_NAME=Node_Name"
    ```

9) In a Bash Shell

    Copy/paste'ing this into my login shell works for me!  _haha!!_  Seriously, though, there is code in `build-coreos-libvirt` for it to create VMs as a regular user, too, but that also depends on sudo to update the files necessary to allow the user full access to the bridge interface.  Basically, you need root.

    ```
    # starting with a bash login shell
    if [ "$USER" != "root" ]; then sudo su -; fi
    mkdir -p /var/lib/libvirt && cd /var/lib/libvirt
    if [ ! -d coreos-libvirt ]; then git clone git@github.com:josephtingiris/coreos-libvirt.git; fi
    cd coreos-libvirt
    ./build-coreos-libvirt kvm-core 3
    virsh start kvm-core01
    virsh start kvm-core02
    virsh start kvm-core03
    # wait a bit for DHCP to nsupdate & etcd cluster to form
    sleep 60 && ssh kvm-core01
    # inside kvm-core01;
    sleep 60 && systemctl is-system-running && etcdctl cluster-health
    sleep 60 && systemctl is-system-running && etcdctl cluster-health
    ```

    Here's what my finished output looks like ... sometimes sleeping longer helps.
    ```
    [jtingiris@dt ~]$ # starting with a bash login shell
    [jtingiris@dt ~]$ if [ "$USER" != "root" ]; then sudo su -; fi
    Last login: Fri Jul  6 17:43:52 EDT 2018 on pts/11

    Next RELEASE is Wed 20180711

    CentOS Linux release 7.5.1804 (Core) 

    /home/jtingiris/.bashrc 20180208, joseph.tingiris@gmail.com [/home/jtingiris/.tmux.conf]

    [root@dt ~]# cd /var/lib/libvirt
    [root@dt /var/lib/libvirt]# if [ ! -d coreos-libvirt ]; then git clone git@github.com:josephtingiris/coreos-libvirt.git; fi
    Cloning into 'coreos-libvirt'...
    Warning: Permanently added 'github.com' (RSA) to the list of known hosts.
    X11 forwarding request failed on channel 0
    remote: Counting objects: 55, done.
    remote: Compressing objects: 100% (32/32), done.
    remote: Total 55 (delta 31), reused 45 (delta 21), pack-reused 0
    Receiving objects: 100% (55/55), 16.59 KiB | 0 bytes/s, done.
    Resolving deltas: 100% (31/31), done.
    [root@dt /var/lib/libvirt]# cd coreos-libvirt
    [root@dt /var/lib/libvirt/coreos-libvirt]# ./build-coreos-libvirt kvm-core 3
    CoreOS_ISO_URL           = https://stable.release.core-os.net/amd64-usr/current/coreos_production_qemu_image.img.bz2{,.sig}

    Downloading coreos_production_qemu_image.img ...

    [1/2]: https://stable.release.core-os.net/amd64-usr/current/coreos_production_qemu_image.img.bz2 --> coreos_production_qemu_image.img.bz2
    --_curl_--https://stable.release.core-os.net/amd64-usr/current/coreos_production_qemu_image.img.bz2
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100  362M  100  362M    0     0  11.0M      0  0:00:32  0:00:32 --:--:-- 11.0M

    [2/2]: https://stable.release.core-os.net/amd64-usr/current/coreos_production_qemu_image.img.bz2.sig --> coreos_production_qemu_image.img.bz2.sig
    --_curl_--https://stable.release.core-os.net/amd64-usr/current/coreos_production_qemu_image.img.bz2.sig
    100   565  100   565    0     0  13385      0 --:--:-- --:--:-- --:--:-- 13385

    Ct_URL                   = https://github.com/coreos/container-linux-config-transpiler/releases/download/v0.9.0/ct-v0.9.0-x86_64-unknown-linux-gnu

    Downloading ct-v0.9.0-x86_64-unknown-linux-gnu ...
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100   624    0   624    0     0   1153      0 --:--:-- --:--:-- --:--:--  1155
    100 6100k  100 6100k    0     0  2311k      0  0:00:02  0:00:02 --:--:-- 3398k

    Etcd_Discovery_URL       = https://discovery.etcd.io/fee2827e548ffd6079e214fb88fbf4b1 (updated)

    Node_Prefix              = kvm-core
    Nodes                    = 3
    RAM                      = 1024
    VCPU                     = 1

    Node_Name                = kvm-core01 (52:54:00:86:6a:e4)
    Disk_Path                = /var/lib/libvirt/coreos-libvirt/vm/kvm-core01 (created)
    Disk_Img                 = /var/lib/libvirt/coreos-libvirt/vm/kvm-core01/kvm-core01.qcow2 (created)
    Ignition_YML             = /var/lib/libvirt/coreos-libvirt/ignition.yml (exists)
    Ignition_Template        = /var/lib/libvirt/coreos-libvirt/ignition-template.json (re-created)
    Ignition_Template        = /var/lib/libvirt/coreos-libvirt/ignition-template.json (exists)
    Ignition_JSON            = /var/lib/libvirt/coreos-libvirt/vm/kvm-core01/ignition.json (created)
    Libvirt_Domain_Xml       = /var/lib/libvirt/coreos-libvirt/vm/kvm-core01/kvm-core01.xml (created)

    Domain kvm-core01 defined from /var/lib/libvirt/coreos-libvirt/vm/kvm-core01/kvm-core01.xml

    virsh start kvm-core01; virsh console kvm-core01
    Node_Name                = kvm-core02 (52:54:00:7d:89:21)
    Disk_Path                = /var/lib/libvirt/coreos-libvirt/vm/kvm-core02 (created)
    Disk_Img                 = /var/lib/libvirt/coreos-libvirt/vm/kvm-core02/kvm-core02.qcow2 (created)
    Ignition_YML             = /var/lib/libvirt/coreos-libvirt/ignition.yml (exists)
    Ignition_Template        = /var/lib/libvirt/coreos-libvirt/ignition-template.json (exists)
    Ignition_JSON            = /var/lib/libvirt/coreos-libvirt/vm/kvm-core02/ignition.json (created)
    Libvirt_Domain_Xml       = /var/lib/libvirt/coreos-libvirt/vm/kvm-core02/kvm-core02.xml (created)

    Domain kvm-core02 defined from /var/lib/libvirt/coreos-libvirt/vm/kvm-core02/kvm-core02.xml

    virsh start kvm-core02; virsh console kvm-core02
    Node_Name                = kvm-core03 (52:54:00:3a:c4:56)
    Disk_Path                = /var/lib/libvirt/coreos-libvirt/vm/kvm-core03 (created)
    Disk_Img                 = /var/lib/libvirt/coreos-libvirt/vm/kvm-core03/kvm-core03.qcow2 (created)
    Ignition_YML             = /var/lib/libvirt/coreos-libvirt/ignition.yml (exists)
    Ignition_Template        = /var/lib/libvirt/coreos-libvirt/ignition-template.json (exists)
    Ignition_JSON            = /var/lib/libvirt/coreos-libvirt/vm/kvm-core03/ignition.json (created)
    Libvirt_Domain_Xml       = /var/lib/libvirt/coreos-libvirt/vm/kvm-core03/kvm-core03.xml (created)

    Domain kvm-core03 defined from /var/lib/libvirt/coreos-libvirt/vm/kvm-core03/kvm-core03.xml

    virsh start kvm-core03; virsh console kvm-core03
    [root@dt /var/lib/libvirt/coreos-libvirt]# virsh start kvm-core01
    Domain kvm-core01 started

    [root@dt /var/lib/libvirt/coreos-libvirt]# virsh start kvm-core02
    Domain kvm-core02 started

    [root@dt /var/lib/libvirt/coreos-libvirt]# virsh start kvm-core03
    Domain kvm-core03 started

    [root@dt /var/lib/libvirt/coreos-libvirt]# # wait a bit for DHCP to nsupdate & etcd cluster to form
    [root@dt /var/lib/libvirt/coreos-libvirt]# sleep 60 && ssh kvm-core01
    Warning: Permanently added 'kvm-core01' (ECDSA) to the list of known hosts.
    X11 forwarding request failed on channel 0
    # inside kvm-core01;
    sleep 60 && systemctl is-system-running && etcdctl cluster-healthContainer Linux by CoreOS stable (1745.7.0)
    kvm-core01 ~ # # inside kvm-core01;
    kvm-core01 ~ # sleep 60 && systemctl is-system-running && etcdctl cluster-health
    starting
    kvm-core01 ~ # sleep 60 && systemctl is-system-running && etcdctl cluster-health
    running
    member 888e715fb65a3c1 is healthy: got healthy result from http://172.22.100.26:2379
    member 76c393bbe963d616 is healthy: got healthy result from http://172.22.100.74:2379
    member b5a355babac0a8ca is healthy: got healthy result from http://172.22.100.75:2379
    cluster is healthy
    kvm-core01 ~ # 
    ```

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

