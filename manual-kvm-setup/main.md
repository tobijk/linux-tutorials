-------------------------------------------------------------------------------

subject: Tutorial
title: Deploying Virtual Machines on Linux
subtitle: A minimalist approach using KVM on Debian 9 <qq>Stretch</qq>
author: Tobias Koch &lt;<tobias.koch@gmail.com>&gt;
date: 2 January 2018

document-type: report
bcor: 0cm
div: 16
lang: en_US
papersize: a4
parskip: half
secnumdepth: 5
secsplitdepth: 0
tocdepth: 5

legal:

 Copyright © 2018 Tobias Koch

 Permission is granted to copy, distribute and/or modify this document under the
 terms of the GNU Free Documentation License, Version 1.3 or any later version
 published by the Free Software Foundation; with no Invariant Sections, no
 Front-Cover Texts, and no Back-Cover Texts.

 The full license text is available at <https://www.gnu.org/licenses/fdl-1.3.en.html>

-------------------------------------------------------------------------------


Prerequisites
===============================================================================

Before we can start running virtual machines, we have to make some preparations
on the host system. For this tutorial we assume that the host OS is a Debian 9
<qq>Stretch</qq> installation.

## Installing Required Packages

Since version 1.3, KVM support has been fully integrated into upstream QEMU. The
easiest way to install QEMU for full x86 system virtualization on Debian is to
install the `kvm` package, which pulls in all dependencies:

```sh
apt-get install kvm
```

## Creating a Dedicated User

We are going to run virtual machines from a non-privileged user account. For
this purpose, create a user *vm* with password logins disabled:

```sh
adduser --disabled-password vm
```

And add it to the *kvm* group:

```sh
adduser vm kvm
```

Adding the user to the *kvm* group gives it access to the `/dev/kvm`
device:

```sh
crw-rw---- 1 root kvm 10, 232 Dez 31 23:30 /dev/kvm
```

Note that we are creating a regular user with home directory `/home/vm`.
Depending on your requirements, you may want to create a system user with a
home directory somewhere below `/var/lib` instead.


Installing a VM
===============================================================================

This section explains how to prepare disk storage, how to configure and start a
virtual machine and how to access its display remotely. You can install any
operating system you like into the VM we are about to create, but instructions
below assume that you want to install a Debian-based OS.

## Downloading Installation Media

Become user *vm*

```sh
su - vm
```

and download Debian's net installer image:

```sh
mkdir -p iso &amp;&amp; cd iso
wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-9.3.0-amd64-netinst.iso
```

## Preparing a Disk Image

For the VM's hard disk, we choose the versatile qcow2 image format. As user
*vm*, in a location of your choice, run

```sh
qemu-img create -fqcow2 -opreallocation=falloc disk1.qcow2 50G
```

This creates a 50GB big, pre-allocated disk image. If you want the disk image to
grow dynamically, you can instead set the pre-allocation mode to *off* or
*metadata*.

## Starting the VM

In the folder where you created your disk image, create a script *start.sh* with
the following contents:

```sh
#!/bin/sh

VM_SPICE_PORT=44401
VM_MAC_ADDR="52:54:01:01:01:01"
VM_BOOT_ORDER=dc
VM_NUM_CPUS=2
VM_MEMORY=1G
VM_DISK_IMG1="disk1.qcow2,format=qcow2"

qemu-system-x86_64 \
    -enable-kvm \
    -smp "cpus=$VM_NUM_CPUS" \
    -m "$VM_MEMORY" \
    -vga qxl \
    -drive "file=$VM_DISK_IMG1,index=0,media=disk,if=virtio,cache=writeback" \
    -spice "addr=127.0.0.1,port=$VM_SPICE_PORT,disable-ticketing" \
    -monitor unix:monitor.sock,server,nowait \
    -daemonize \
    -pidfile pid \
    -boot "order=$VM_BOOT_ORDER" \
    -cdrom ../../iso/debian-9.3.0-amd64-netinst.iso
```

This instructs QEMU to start the virtual machine with 2 CPUs, 1GB of memory and
boot from CD-ROM, which is boot device d. QEMU will also write a pid file and
fork into the background. Execute this script as user *vm* and use *ps* to
verify that QEMU is indeed running.

## Accessing the Display via SPICE

In the start script, we enabled the SPICE protocol on port `44401`, which gives
us remote access to the virtual machine's display and lets us control keyboard
and mouse. For security reasons, we have bound the SPICE server to the loopback
interface only. In order to access it securely from a remote machine, we can
use port-forwarding via SSH:

```sh
ssh -L44401:127.0.0.1:44401 user@vm-server.example.com
```

This opens port `44401` on the client side and forwards it through the SSH
tunnel to address `127.0.0.1` port `44401`, which is the loopback interface on
the server itself.

On the client, install a SPICE client application such as *spicy* and connect
to `localhost` port `44401`. You should now see the Debian installer's boot
menu. Continue with installing the operating system, then proceed to the next
section in this manual.

<title>Note:</title> If you intend to install a graphical user interface inside
your virtual machine, you might want to install and enable the SPICE vdagent
inside the virtal machine.

## Booting from Harddisk

When you restart your VM after finishing the installation, it will again boot
into the Debian installer. In order to boot from harddisk instead, you need to
fix the boot order in the *start.sh* script. Simply removing the *d* from
`VM_BOOT_ORDER` will do the trick. But we also need to re-launch QEMU for the
new setting to take effect.


VM Management
===============================================================================

Now we have successfully launched a VM and even installed an operating system,
but how do we shut it down without data loss and how do we launch it
automatically when the host boots up?

## Controlled Shutdown

While the virtual machine shows up as an ordinary process on the host system, we
cannot simply terminate that process. We need a way to cleanly shut down the
guest operating system instead. QEMU has a built-in monitoring console, which we
connected to a unix socket represented in the file system as *monitor.sock*.

In order to connect to the socket and gain access to the console, you can use
the *socat* program:

```sh
socat - UNIX-CONNECT:monitor.sock
```

Once logged on to the monitoring console, you can send the guest operating
system an ACPI shutdown event via

```sh
(qemu) system_powerdown
```

We add just a tad more logic and wrap this into a corresponding *stop.sh*
script:

```sh
#!/bin/sh

if [ ! -f pid ]; then
    exit 0
fi

VM_PID="`cat pid`"

for i in seq 1 12; do
    echo system_powerdown | socat - UNIX-CONNECT:monitor.sock \
        >/dev/null 2>&1

    for i in seq 1 5; do
        if [ ! -d /proc/$VM_PID ]; then
            rm -f pid
            break
        fi

        sleep 1
    done

    if [ ! -f pid ]; then
        break
    fi
done

if [ -f pid ]; then
    kill -TERM $VM_PID 2>/dev/null
    sleep 5
    kill -KILL $VM_PID 2>/dev/null
fi

exit 0
```

## Launching VMs During Boot

Now that we're able to start and stop VMs properly, we want to automate the
whole thing through an init script. The following will work just fine also with
systemd's SysVinit emulation:

```sh
#! /bin/sh

### BEGIN INIT INFO
# Provides:          vm
# Required-Start:    $remote_fs $syslog
# Required-Stop:     $remote_fs $syslog
# Should-Start:      $named autofs
# Default-Start:     2 3 4 5
# Default-Stop:      
# Short-Description: starts KVM virtual machines
# Description:       this startup script launches all virtual machines in 
#                    /home/vm/vm
### END INIT INFO

set -e

. /lib/lsb/init-functions

VM_BASE_DIR=/home/vm/vm

case "$1" in
  start)
        log_daemon_msg "Starting virtual machines" ""
        for VM_DIR in $VM_BASE_DIR/*
        do
                if [ ! -f "$VM_DIR/start.sh" ]; then
                        continue
                fi

                VM_NAME="`basename $VM_DIR`"
                log_progress_msg "$VM_NAME"
                (cd $VM_DIR && su -c "$VM_DIR/start.sh" vm)
        done
        log_end_msg 0
        ;;
  stop)
        log_daemon_msg "Stopping virtual machines" ""
        for VM_DIR in $VM_BASE_DIR/*
        do
                if [ ! -f "$VM_DIR/stop.sh" ]; then
                        continue
                fi

                VM_NAME="`basename $VM_DIR`"
                log_progress_msg "$VM_NAME"
                (cd $VM_DIR && su -c "$VM_DIR/stop.sh" vm)
        done
        log_end_msg 0
        ;;

  reload|force-reload)
        log_warning_msg "Reloading VMs is not supported."
        ;;

  restart)
        $0 stop
        $0 start
        ;;

  status)
        for VM_DIR in $VM_BASE_DIR/*
        do
                if [ ! -f "$VM_DIR/start.sh" ]; then
                        continue
                fi

                VM_NAME="`basename $VM_DIR`"

                if [ -f "$VM_DIR/pid" ]
                then
                        VM_PID="`cat $VM_DIR/pid`"
                        if [ -d "/proc/$VM_PID" ]
                        then
                                VM_STATE="up"
                        else
                                VM_STATE="down (stale)"
                        fi
                else
                        VM_STATE="down"
                fi

                echo "* $VM_NAME: $VM_STATE"
        done
        ;;
  *)
        echo "Usage: /etc/init.d/vm {start|stop|reload|force-reload|restart|status}"
        exit 1
esac

exit 0
```

Adapt the `VM_BASE_DIR` parameter to your own setup and install the script to
`/etc/init.d/vm`. Now enable the service via

```sh
systemctl enable vm
```


Networking
===============================================================================

Chances are high that you want to run VMs because you need to partition a number
of network services. In this section we will configure connectivity for our VMs.

## Creating a TAP interface

For each virtual machine, we need to create a TAP device, which will allow us
to establish communication between the VM and the host. The TAP device is a
virtual link of which one end is terminated on the host and the other end
inside the VM.

To create a TAP interface, run the `ip` command from the `iproute2` package as
root:

```sh
ip tuntap add mode tap user vm group vm name tap0
```

Each time we create a new TAP device, we have to give it a different name. We
also need to specify the user and group the TAP device will belong to, in order
to ensure that the `vm` user can open them.

## Bridged Networking

Now that we know how to create virtual network links for our VMs, we want to
connect them using some sort of virtual network switch. The easiest way to
achieve this is by adding all relevant interfaces on a virtual network bridge.

Bridges are created and managed with the `brctl` command from the `bridge-utils`
package.

```sh
brctl addbr br0
brctl addif br0 tap0
...
```

We can automate all of this by adding the following entries to
`/etc/network/interfaces`:

```
auto br0
iface br0 inet static
    pre-up ip tuntap add mode tap user vm group vm name tap0 || true
    pre-up ip tuntap add mode tap user vm group vm name tap1 || true
    bridge_ports tap0 tap1
```

If you have public IP addresses for all your VMs, you can connect them to the
outside world by adding a physical interface, such as `eth0`, to the
`bridge_ports` list. If that physical interface has an address assigned, you
will need to *remove* it from the physical interface and assign it to the
*bridge* interface `br0`, instead.

## <label id="section:routed_networking"/> Routed Networking

If you do *not* have public IP addresses to assign to your VMs, you can instead
create a private network on the bridge interface and then forward traffic via
NAT or with a proxy to your virtual machines.

For this setup, we first extend our interfaces configuration a little bit:

```
auto br0
iface br0 inet static
    pre-up ip tuntap add mode tap user vm group vm name tap0 || true
    pre-up ip tuntap add mode tap user vm group vm name tap1 || true
    bridge_ports tap0 tap1
    address 192.168.0.1
    netmask 255.255.255.0

iface br0 inet6 static
    address fd00:0:0:1::1/64
```

Note that we also assigned a private IPv6 address from the fd00::/8 range,
because in the modern world, you do want to support IPv6.

Before we go on with NAT and port-forwarding, we need to take care of the
network configuration inside of our VMs.

## Fixing the VM Configuration

In order to connect a virtual machine to one of the TAP devices, you just need
to add a few lines to the `start.sh` script:

```sh
#!/bin/sh

VM_SPICE_PORT=44401
VM_MAC_ADDR="52:54:01:01:01:01"
VM_BOOT_ORDER=c
VM_NUM_CPUS=2
VM_MEMORY=1G
VM_DISK_IMG1="disk1.qcow2,format=qcow2"
VM_TAP_DEV=tap0

qemu-system-x86_64 \
    -enable-kvm \
    -smp "cpus=$VM_NUM_CPUS" \
    -m "$VM_MEMORY" \
    -vga qxl \
    -drive "file=$VM_DISK_IMG1,index=0,media=disk,if=virtio,cache=writeback" \
    -spice "addr=127.0.0.1,port=$VM_SPICE_PORT,disable-ticketing" \
    -monitor unix:monitor.sock,server,nowait \
    -device "virtio-net-pci,mac=$VM_MAC_ADDR,netdev=$VM_TAP_DEV" \
    -netdev "tap,id=$VM_TAP_DEV,ifname=$VM_TAP_DEV,script=no,downscript=no" \
    -daemonize \
    -pidfile pid \
    -boot "order=$VM_BOOT_ORDER"
```

Next time you boot into the VM, you will see a new ethernet interface ready to
be configured. If your VM is going to have a public IP address, go ahead and
assign that. If you are going with a private network setup using NAT, then you
can go with a setup like this, in `/etc/network/interfaces` on your VM:

```
# The primary network interface
allow-hotplug ens3
iface ens3 inet static
    address 192.168.0.102/24
    gateway 192.168.0.1
    # dns-* options are implemented by the resolvconf package, if installed
    dns-nameservers X.X.X.X Y.Y.Y.Y
    dns-search example.com

iface ens3 inet6 static
    address fd00:0:0:1::102/64
    gateway fd00:0:0:1::1
```

This configuration will put your VM in the private network that we created
in section <ref idref="section:routed_networking"/>. After bringing it up, you
should already be able to ping the gateway `192.168.0.1`.

## Hiding behind a NAT

If you went with a private network setup and now want to expose some of the
services running inside your VMs to the Internet, you can create a combination
of port-forwarding and network address translation.

In the following example, we masquerade the private network we created before,
which effectively allows Internet access *from* the VMs to the outside world.
In order to additionally expose a number of services, such as SSH and HTTP,
we create port-forwardings from the host to the corresponding VMs.

We can do this for IPv4 and IPv6 analogously:

```sh
#!/bin/sh

IF_INTERNET="enp4s0"
IF_VM_NET="br0"

###############################################################################
# IPV4
###############################################################################

iptables -t filter -X
iptables -t filter -F
iptables -t nat -X
iptables -t nat -F

# enable IP forwarding between VMs and WWW
iptables -t filter -A FORWARD -i $IF_VM_NET -o $IF_INTERNET -j ACCEPT
iptables -t filter -A FORWARD -o $IF_VM_NET -i $IF_INTERNET -j ACCEPT
iptables -t filter -A FORWARD -j DROP

# source NAT for VMs to reach the outside world
iptables -t nat -A POSTROUTING -o $IF_INTERNET -s 192.168.0.0/24 -j MASQUERADE

# destination NAT for services
iptables -t nat -p tcp -A PREROUTING -i $IF_INTERNET --dport 80    \
	-j DNAT --to 192.168.0.101:80
iptables -t nat -p tcp -A PREROUTING -i $IF_INTERNET --dport 22101 \
	-j DNAT --to 192.168.0.101:22
iptables -t nat -p tcp -A PREROUTING -i $IF_INTERNET --dport 22102 \
	-j DNAT --to 192.168.0.102:22

echo 1 > /proc/sys/net/ipv4/ip_forward

###############################################################################
# IPV6
###############################################################################

ip6tables -t filter -X
ip6tables -t filter -F
ip6tables -t nat -X
ip6tables -t nat -F

# enable IP forwarding between VMs and WWW
ip6tables -t filter -A FORWARD -i $IF_VM_NET -o $IF_INTERNET -j ACCEPT
ip6tables -t filter -A FORWARD -o $IF_VM_NET -i $IF_INTERNET -j ACCEPT
ip6tables -t filter -A FORWARD -j DROP

# source NAT for VMs to reach the outside world
ip6tables -t nat -A POSTROUTING -o $IF_INTERNET -s fd00:0:0:1::/64 -j MASQUERADE

# destination NAT for services
ip6tables -t nat -p tcp -A PREROUTING -i $IF_INTERNET --dport 80    \
	-j DNAT --to [fd00:0:0:1::101]:80
ip6tables -t nat -p tcp -A PREROUTING -i $IF_INTERNET --dport 22101 \
	-j DNAT --to [fd00:0:0:1::101]:22
ip6tables -t nat -p tcp -A PREROUTING -i $IF_INTERNET --dport 22102 \
	-j DNAT --to [fd00:0:0:1::102]:22

echo 1 > /proc/sys/net/ipv6/conf/all/forwarding
```

Make sure an adaptation of this script runs when the host boots up and you are
good to go.

On a side note: there is no shortage of IPv6 addresses and chances are high that
your provider gave you a block of public IPv6 addresses together with the
machine you rented. You might want to consider using these instead of using
port-forwarding and NAT for IPv6.

