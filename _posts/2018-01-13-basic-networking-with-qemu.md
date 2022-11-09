---
layout: post
title: "Basic networking with QEMU"
permalink: "/basic-networking-with-qemu/"
---

If you're coming to QEMU from a different virtualization platform, the networking bits might seem confusing at first. The [official networking documentation](https://wiki.qemu.org/Documentation/Networking) is far from complete, and it takes some sifting through different sources of information before you can get a decent grasp on the topic.

I'd like to summarize here the basic information required to quickly and painlessly get the network configuration out of the way, so you can concentrate on whatever task you're actually working on.

Generally, **user networking** and **bridged networking** modes should suffice for most use cases, and this is what the rest of this post will be dealing with. Be aware that the *"bridged networking mode"* is referred to as *"tap networking backend"* within the QEMU documentation.
 

## User networking

This is the default networking mode. It requires no configuration on the VM host, no additional privileges, and often no passing of network-related command-line arguments to the QEMU executable. If your guest is set to automatically configure its network interface through DHCP, the following invocation of QEMU should be sufficient for this mode:

```
$ qemu-system-x86_64 -enable-kvm -nographic -hda ./vdisk.img
```

By default, your guest will have the following view of the network:

- 10.0.2.15/24 - Guest's IP address
- 10.0.2.2/24 - Gateway/VM host
- 10.0.2.3/24 - DNS resolver
- 10.0.2.4/24 - SMB server (*Optional. Requres SMB server on the host system*)

Your guest should also be able to connect to any network your VM host has access to, unless there are firewall rules preventing it. It's important to keep in mind that in this mode your guest can only communicate over TCP and UDP protocols. ICMP-based utilities such as `ping` and `traceroute` will not be functional, and therefore unfit for network connectivity tests.

Despite being able to access network resources over TCP and UDP protocols, the guest itself is not directly accessible unless port forwarding is put in place. It can be enabled through the `hostfwd` option to the `-netdev` command-line argument. The syntax is the following:

```
hostfwd=[tcp|udp]:[hostaddr]:hostport-[guestaddr]:guestport
```

For example, in order to access the guest from your VM host over SSH, you would forward the port 2222/tcp on the VM host to the port 22/tcp on the guest in the following manner:

```
$ qemu-system-x86_64 -enable-kvm -nographic -hda ./vdisk.img \
                     -netdev user,id=net0,hostfwd=tcp::2222-:22 \
                     -device e1000,netdev=net0
```

Assuming your guest performs automatic network configuration through DHCP and it starts its SSH daemon upon boot, you should be able to connect to it:

```
$ ssh -p 2222 localhost
```

In order to forward multiple ports, the `hostfwd` option can be specified multiple times.

***Note:** You can skip the network-related command-line parameters only when you want the default user-mode network configuration. If you enable any non-default features, such as port forwarding, you're expexpected to specify both the network backend information through the `-netdev` argument and the network device driver details through the `-device` argument. Failure to do so will render your guest without network capabilities.*{:.warning}

Another thing to keep in mind is that the user networking mode relies on a network stack that is entirely implemented in userspace, within the QEMU process. This has a major impact on guest's network performance.


## Bridged networking with qemu-bridge-helper

You should use bridged networking mode for your QEMU guests if you're dealing with any the following scenarios:

- The network performance aspect of your guests is important
- You're working with network protocols beyond TCP and UDP
- Your guests must be reachable from external networks

Since version 1.1, QEMU ships with the `qemu-bridge-helper` utility, which greatly simplifies the configuration of bridged networking for your guests.

In order to enable bridged networking for your QEMU guests, you must first create and configure a bridge interface on your host. For example, Ubuntu 17.04 uses `netplan` for network configuration. In order to configure a statically-addressed bridge interface `br0` consisting of your ethernet interface `enp0s25`, the netplan configuration file (`/etc/netplan/01-netcfg.yaml`) would resemble the following:

```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s25:
      dhcp4: false
  bridges:
    br0:
      interfaces: [enp0s25]
      addresses:
          - 192.168.1.2/24
      gateway4: 192.168.1.254
      nameservers:
        search: [home]
        addresses: [8.8.8.8, 8.8.4.4]
      parameters:
        stp: false
        forward-delay: 0
```

In order for the configuration to take effect, it must be applied with the `netplan` utility:

```
$ sudo netplan --debug apply
```

This is a OS/distribution-specific task, so you'll have to refer to your platform's documentation for more details.

Next, you must specify the newly-created bridge interface in `/etc/qemu/bridge.conf`:

```
$ sudo mkdir /etc/qemu
$ sudo touch /etc/qemu/bridge.conf && sudo chmod 644 /etc/qemu/bridge.conf
$ sudo sh -c "echo 'allow br0' >> /etc/qemu/bridge.conf"
```

Finally, in order for non-privileged processes to be able to invoke `qemu-bridge-helper`, you must set the SETUID bit on the utility:

```
$ sudo chmod u+s /usr/local/libexec/qemu-bridge-helper
```
***Note:** Be aware that by setting the SETUID bit on a root-owned binary, you're exposing your system to a potential privilege escalation attack in case the aforementioned binary is affected by a security issue. Proceed with caution.*{:.warning}

You should now be able to invoke QEMU as an unpriviledged user and make use of the bridged networking backend:

```
$ qemu-system-x86_64 -enable-kvm -nographic -hda ./vdisk.img \
                     -netdev bridge,id=net0,br=br0 -device e1000,netdev=net0
```

If your guest is set to configure networking through DHCP, it should have acquired an address from your network's DHCP server.


**Further reading:**
- [QEMU Networking documentation](https://wiki.qemu.org/Documentation/Networking)
- [qemu-bridge-helper documentation](https://wiki.qemu.org/Features/HelperNetworking)
- [QEMU Networking on Wikibooks](https://en.wikibooks.org/wiki/QEMU/Networking)
- [Extensive QEMU documentation on ArchLinux wiki](https://wiki.archlinux.org/index.php/QEMU)
