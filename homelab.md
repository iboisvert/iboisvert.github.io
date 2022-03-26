# Home Lab
## What?
NAS, DHCP, caching DNS, and other misc services like git.

## Why?
I bought a Dell T710 on the cheap over a year ago. I've been using FreeBSD with some success to run services like NAS, DHCP, and caching DNS, but I want to run a bunch of other services as well like Minecraft and Ubiquiti. In FreeBSD it would make sense to run these services in jails for isolation, but the CLI for managing jails is pretty clunky. I am using `ez-jail` but it seems to be unsupported.

Honestly though, I just wanted to experiment with a hypervisor.

# Host

Summary:  

<table>
<thead><td>Machine</td><td>OS</td><td>Services</td><td>Specs</td></thead>
<tr><td>Host</td><td>Ubuntu Server 20.04 LTS</td><td>SSH</td><td>Dell PowerEdge T710, 16GB, 4x BCM5709 GB nic, LSI SAS2008 HBA</td></tr>
<tr><td>VM 1&mdash;gateway</td><td>OpenBSD 7.0</td><td>NAT, firewall, DHCP, caching DNS, NTP</td><td>1G, 1 CPU, 2 MACVTAP nic (wan-private, lan-bridge)</td></tr>
<tr><td>VM 2&mdash;nas</td><td>OpenBSD 7.0</td><td>Samba</td><td>1G, 1 CPU, 1 MACVTAP nic (bridge)</td></tr>
<tr><td>VM 3&mdash;cloud</td><td>Ubuntu Server 20.04 LTS</td><td>Docker</td><td>2G, 2CPU, 1 MACVTAP nic (bridge)</td></tr>
</table>

The host will run Linux because [KVM](https://www.linux-kvm.org/page/Main_Page) is baked in to the Linux kernel and [libvirt](https://libvirt.org/index.html) `virsh` is a really good management interface. [Ubuntu Server](https://ubuntu.com/)&nbsp;20.04&nbsp;LTS is used as the host OS because [CentOS Stream](https://www.centos.org/centos-stream/)&nbsp;8 dropped (or deprecated, doesn't matter) support for both the BCM5709 and SAS2008.

Windows Server would probably have worked really well for this application, but I didn't consider it because I don't want to be a Windows sysadmin.

I investigated Xen on NetBSD and bhyve on FreeBSD.

As a side note, I initially tried setting up the host OS on a disk connected to the USB port of my laptop. The USB boot disk became corrupted once when waking from sleep. Apparently using USB for a spinning rust boot disk may not be a great idea.

## Mirror Host Disk

I'm going to use software RAID because I feel that it is safer to use software RAID than hardware RAID provided by a card over a decade old: if the card dies it will take at least two weeks to get an eBay replacement; if the machine dies, I cannot pull the disks and drop into a different machine without also pulling the HBA card.

[man mdadm](https://manpages.ubuntu.com/manpages/trusty/en/man8/mdadm.8.html)

Ubuntu Server doesn't allow a degraded RAID 0 mirror to be created on install, so the disk needs to be partitioned before installing the host OS. I cannot remember exactly how I did this, probably by switching consoles while installing.

<ol>
<li>Create these partitions:</li>
   <ol>
   <li>600 MB EFI /boot/efi</li>
   <li>1 GB vfat /boot</li>
   <li>remaining LVM</li>
      <ol type="a">
         <li>4 GB swap</li>
         <li>256 GB XFS /home</li>
         <li>remaining XFS / (root)</li>
      </ol>
   </ol>
</ol>

XFS file systems cannot be shrunk. This **will** cause future issues if you are not careful about partitioning the host when installing.

Now, since it is easier to resize LVs after creating, resize the root LV:

```lvresize -l -1 {LV}```

to provide space for the mdadm metadata that will be stored at the end of partition 2.

## Install Host OS

1. Download Ubuntu Server LTS 20.04. Install onto partitions created in previous section

2. Disable cloud-init  
   ```
   # systemctl disable cloud-init
   ```

3. Install 
   ```
   # apt install qemu-kvm libvirt-daemon-system virtinst libosinfo-bin
   ```

## Networking

The gateway will handle all intranet traffic to the WAN, so it is not necessary for the WAN network device on the host to have an IP address--the WAN network device can be left unconfigured. On the LAN side, it will be necessary to create a MACVTAP device so that the host can talk to the VMs*. 

Netplan doesn't support configuring MACVTAP devices, so the device will be configured using systemd-networkd. Why Canonical chose to use Netplan in Ubuntu Server is anybody's guess...

Here is the Netplan configuration, all devices are unconfigured:  

`/etc/netplan/00-installer-config.yaml`  
```
network:
  version: 2
```

And here is the systemd network files:

`/etc/systemd/network/10-lan.network`
```
[Match]
Name=$LAN_IFACE

[Network]
MACVTAP=macvtap0
```

`/etc/systemd/network/10-macvtap0.netdev`
```
[Match]

[NetDev]
Name=macvtap0
Kind=macvtap

[MACVTAP]
Mode=bridge
```

`/etc/systemd/network/11-macvtap0.network`
```
[Match]
Name=macvtap0

[Link]
MACAddress=9e:ce:0a:18:97:0a

[Network]
Address=172.27.0.1/20  # Set static address
DHCP=yes  # Attempt to get address from DHCP
LinkLocalAddressing=ipv6
```

Note that the filenames here are important. As per the systemd.network man page: <q cite="https://manpages.ubuntu.com/manpages/bionic/man5/systemd.network.5.html">The first (in lexical order) of the network files that matches a given device is applied, all later files are ignored, even if they match as well.</q> Ubuntu creates a `.network` file in `/usr/lib/systemd/network` that configures the host adapters and will conflict with user-defined configurations**.

Both `Address` and `DHCP` keys are present in the `11-macvtap0.network` file. I hope that this means that the device will be set to the given static IP if DHCP fails, but I haven't tested it.

\* It seems that my network switch doesn't support hairpin mode  
** I might have removed the conflicting file because I can't find it now.

# VM Guest Config

## libvirt Storage Pool

It isn't necessary to create a storage pool for VMs, but with a storage pool all volumes (disk images) can easily be managed in the same location.  

The alternative to using a storage pool would be to create disk images for every VM in the directory in which the VM config resides. In this case, `libvirt` will automagically create a storage pool for every VM and the storage pools for every VM will need to be managed independently.

It's not yet clear which method is preferred.

7. To create the storage pool:
   ```
   # mkdir /var/lib/vm-images
   # virsh pool-create-as vm-images dir --target /var/lib/vm-images
   ```

## OpenBSD

OpenBSD will serve as the OS for the gateway and NAS. Before OpenBSD can be installed, we need to modify the ISO image to boot using a serial console so that we don't need to use VNC:

1. Download [OpenBSD](https://ftp.openbsd.org/pub/OpenBSD/7.0/amd64/install70.iso)

2. Write a new boot configuration to the ISO image to enable the serial console:
   ```
   # mount -o ro install70.iso /mnt
   $ cp /mnt/etc/boot.conf ./boot.conf
   # umount /mnt
   $ echo set tty com0 >> ./boot.conf
   $ genisoimage -o install70-tty_com0.iso -M install70.iso -l -R -graft-points /etc/boot.conf=boot.conf
   ```

## 2. Build Gateway

OpenBSD is used for the gateway for a single main reason: I trust OpenBSD. Some other reasons: 
- The default install runs only a handful of daemons
- pf firewall configuration is simple
- ufw/iptables rules are an absolute mess that is impossible to follow.

2. Create and start gateway VM:
   ```
   # virsh vol-create-as vm-images gateway-disk0.qcow2 10G
   # virt-install 
      --name gateway 
      --os-variant openbsd6.6
      --memory 1024
      --vcpus 1
      --cdrom install70-tty_com0.iso
      --disk vol=vm-images/gateway-root.qcow2
      --disk path=/var/lib/iso/install70.img,format=raw,readonly=on 
      --network type=direct,source=$LAN_IFACE,source.mode=bridge  # LAN
      --network type=direct,source=$WAN_IFACE,source.mode=private  # WAN
      --graphics none 
      --serial pty 
      --console pty,target_type=serial
   ```

3. At bootloader menu:
   ```
   boot> set tty com0
   boot> boot
   ```
   Follow prompts to install OpenBSD

4. Create `doas.conf`:
   ```
   # echo permit persist :wheel > /etc/doas.conf
   ```

4. Configure network devices:  
   `/etc/hostname.vio0`
   ```
   inet autoconf
   group egress
   ```

   `/etc/hostname.vio1`
   ```
   inet 172.27.0.2 0xfffff000
   group ingress
   ```

5. Enable forwarding between ports
   ```
   # echo 'net.inet.ip.forwarding=1' >> /etc/sysctl.conf
   ```

6. Install [dnsmasq](https://dnsmasq.org/): `pkg_add dnsmasq`. OpenBSD ships with [Unbound](https://man.openbsd.org/unbound) and [dhcpd](https://man.openbsd.org/dhcpd) but dnsmasq is preferred because it includes both DNS and DHCP services and DHCP updates DNS records without having to separately configure `isc-dhcpd` and `named`.

   `dnsmasq.conf`
   ```
   user=_dnsmasq
   group=_dnsmasq
   interface=vio0
   bind-interfaces  # Bind only to interfaces declared by interface=
   domain-needed
   bogus-priv
   dhcp-authoritative
   dhcp-leasefile=/var/db/dnsmasq/dnsmasq.leases
   no-resolv  # Do not read nameservers from /etc/resolve.conf
   # CIRA nameservers
   server=149.112.121.20
   server=149.112.122.20
   # Cloudflare
   server=1.1.1.1

   domain=home.arpa
   dhcp-range=172.27.1.16,172.27.1.255,255.255.240.0,1w
   dhcp-name-match=set:wpad-ignore,wpad
   dhcp-ignore-names=tag:wpad-ignore
   dhcp-option=option:router,172.27.0.1
   #dhcp-option=option:ntp-server,172.27.0.1

   # Define static IPs here
   #dhcp-host=11:22:33:44:55:66,fred,192.168.0.60,45m
   ```

7. Instructions in [PF FAQ](http://www.openbsd.org/faq/pf/example1.html) to build a router with DHCP and caching DNS.  
   `/etc/pf.conf`
   ```
   lan0="vio1"
   wan0="vio0"

   table <martians> { 0.0.0.0/8 10.0.0.0/8 127.0.0.0/8 169.254.0.0/16     \
            172.16.0.0/12 192.0.0.0/24 192.0.2.0/24 224.0.0.0/3 \
            192.168.0.0/16 198.18.0.0/15 198.51.100.0/24        \
            203.0.113.0/24 }

   set block-policy drop
   set loginterface $wan0
   set skip on lo

   match in all scrub (no-df random-id max-mss 1440)
   match out on egress inet from !(egress:network) to any nat-to (egress:0)

   antispoof quick for { egress $lan0 }

   block in quick on egress from <martians> to any
   block return out quick on egress from any to <martians>

   block return in log on $lan0 from no-route to any
   block return in log on $lan0 from urpf-failed to any

   #block return	# block stateless traffic
   block in on egress

   # By default, do not permit remote connections to X11
   block return in on ! lo0 proto tcp to port 6000:6010

   # Port build user does not need network
   block return out log proto {tcp udp} user _pbuild

   pass in on $lan0 proto tcp from $lan0:network to $lan0:0 port 22  # ssh
   pass in on $lan0 proto udp from $lan0:network to $lan0:0 port 53  # dns
   pass in on $lan0 proto udp from any to $lan0:broadcast port 67  # dhcp

   pass in on $lan0 from any to !$lan0:network  # internet

   pass out quick inet
   ```

8. Set gateway VM to autostart:
   ```
   virsh autostart gateway
   ```

**TODO** Check system memory after all servers installed, maybe reduce memory allocated to VM

## 3. Build NAS

NAS services: Samba

3. Create NAS VM:
   ```
   # virsh vol-create-as vm-images nas-root.qcow2 10G  # Mount /
   # virsh vol-create-as vm-images nas-home.qcow2 256G # Mount /home
   # virsh vol-create-as vm-images nas-srv.qcow2 2T --allocation 500G # Mount /srv
   # virt-install
      --name nas
      --os-variant openbsd6.6
      --memory 1024
      --vcpus 1
      --cdrom install70-tty_com0.iso
      --disk vol=vm-images/nas-disk0.qcow2
      --disk vol=vm-images/nas-disk1.qcow2
      --disk vol=vm-images/nas-disk1.qcow2
      --network type=direct,source=$LAN-IFACE
      --graphics none
      --serial pty 
      --console pty,target_type=serial
   ```

   Mount `sd1` on `/home`

   Note: OpenBSD Samba requires X, either we want to install the `x*` file sets, or build Samba from ports to exclude X dependency. `games*` and `comp*` can be excluded from install.

   After installation, as before, configure `doas`:
   ```
   # echo permit persist <user> > /etc/doas.conf
   ```

5. Mount sd2 on `/srv`

4. Install and configure Samba:
   ```
   # pkg_add samba
   ```

   The installer will report some ambiguity in the `cyrus-sasl` dependency of `openldap-client`. Choose `cyrus-sasl-...-db4` for Berkley DB or `cyrus-sasl-...-sqlite3` for sqlite3. Note that some other dependency of `samba` requires `sqlite` so it might be best to choose `cyrus-sasl-...-sqlite3`.

8. Set nas VM to autostart:
   ```
   virsh autostart nas
   ```

## 4. Build Container Host--the **Cloud**

Instead of hacking the Ubuntu Server installer ISO to support the serial console on boot, I tried booting using UEFI. It seems that UEFI enables both the local console and the serial console on boot, so it is not necessary to hack the GRUB config.

1. Install Ubuntu Server
   ```
   # virsh vol-create-as vm-images cloud-root.qcow2 10G  # Mount /
   # virsh vol-create-as vm-images cloud-var_lib.qcow2 500G  # Mount /var/lib
   # virt-install
      --name cloud
      --os-variant ubuntu20.04
      --memory 2048
      --vcpus 2
      --cdrom CentOS-Stream-*.iso
      --disk vol=vm-images/cloud-root.qcow2
      --disk vol=vm-images/cloud-var_lib.qcow2
      --network type=direct,source=$LAN_IFACE,source.mode=bridge
      --boot uefi
      --graphics none
      --serial pty 
      --console pty,target_type=serial
   ```

At the GRUB menu, edit the default option to install Ubuntu Server. Add these options to the kernel command line:
```
console=ttyS0
```

Proceed with install as usual. Note that there is no benefit to using LVM in this context, so do not select it when allocating the disks for install

2. [Install docker](https://docs.docker.com/engine/install/ubuntu/) or [Install podman](https://podman.io/getting-started/installation.html). Note that if podman is installed, then containers will have to be configured to be started by systemd, but podman provides support for this. 

I installed `jacobalberty/unifi:latest` from [Docker Hub](hub.docker.com) and had some problems accessing the web server. I removed podman, installed docker, and everything seems to work. 

### Install Unifi Network

Formerly Unifi Controller

```
docker run -d --restart=unless-stopped --init -p 8080:8080 -p 8443:8443 -p 3478:3478/udp -p 10001:10001/udp -p 1900:1900/udp -e TZ='America/Edmonton' -v /var/lib/unifi:/unifi --name unifi jacobalberty/unifi:latest
```

See the [unifi-docker](https://github.com/jacobalberty/unifi-docker) project for details.

### Install mindlna

Minidlna serves media from the NAS.

```
$ git clone https://github.com/iboisvert/minidlna-docker.git
$ # Edit minidlna.conf
# ./docker-build.sh
# docker run -d --restart=unless-stopped --init -p 8200:8200/tcp -p 1900:1900/udp -v /var/lib/minidlna:/var/lib/minidlna -v /srv:/srv --name minidlna iboisvert/minidlna:1
```

See the [minidlna-docker](https://github.com/iboisvert/minidlna-docker) project for details.

# References
1. Introduction to Linux interfaces for virtual networking <https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking>
2. [Wishlist] Support macvlan/macvtap interfaces <https://bugs.launchpad.net/netplan/+bug/1664847>
2. Man systemd.network <https://manpages.ubuntu.com/manpages/focal/man5/systemd.network.5.html>
3. Virtualization-libvirt <https://ubuntu.com/server/docs/virtualization-libvirt>
2. Assign individual NIC to KVM guest <https://serverfault.com/a/446664>
3. (Netplan) Support macvlan/macvtap interfaces <https://bugs.launchpad.net/netplan/+bug/1664847>
4. Dell disables SR-IOV on network devices <https://community.intel.com/t5/Ethernet-Products/Sr-IOV-Server-2012-I350-T4/td-p/218188?start=0&tstart=0>
