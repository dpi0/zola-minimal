---
title: Another Page
date: 2023-11-17
---

## `KVM`

Kernel-based Virtual Machine, is a hypervisor built into the Linux kernel

![image](https://i.ibb.co/7VP0b22/kvm.png)

check if virutalization is enabled or not, greater than 0 yes, if not then change from BIOS: `grep -Ec '(vmx|svm)' /proc/cpuinfo`

further checks include: `lscpu | grep Virtualization` I'm using an Intel processor. So I'm getting VT-x as output. If you have an AMD processor, the output should be AMD-V. If the output is blank, the hardware virtualization extension is most likely disabled in the BIOS/UEFI. Before proceeding, enable it. It will be located in the CPU configuration section.

ensure that your kernel includes KVM modules. The module is only available if it is set to y or m. If both steps are successful, proceed with the installation of KVM.

for arch: `zgrep CONFIG_KVM /proc/config.gz`

for other distros: `zgrep CONFIG_KVM /boot/config-$(uname -r)`

on ubuntu/debian

```bash linenums="1"
sudo apt install qemu-system-x86 libvirt-daemon-system virtinst \
virt-manager virt-viewer ovmf swtpm qemu-utils guestfs-tools \
libosinfo-bin tuned
```

```bash linenums="1"
sudo pacman -S qemu-full virt-manager virt-viewer dnsmasq bridge-utils libguestfs ebtables vde2 openbsd-netcat

sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER

# logout

# for using/creating virtual machines
sudo systemctl start libvirtd
sudo virsh net-start default
sudo systemctl restart nftables.service # useful when networking issues occur
```

**set the permissions either ///system or ///user, for basic testing and not worrying too much about security use ///system**

```bash linenums="1"
sudo cp -rv /etc/libvirt/libvirt.conf ~/.config/libvirt/libvirt.conf
sudo chown dpi0:dpi0 ~/.config/libvirt/libvirt.conf
# uncomment: uri_default = "qemu:///system"

virsh list --all

# without above virsh looks at ///user and virt-manager by default runs in ///system
# so to bring virsh to the ///system level we have to use above
```

**and only now you can use virsh normally**

!!! note

    if for some reason KVM fails to connect to the internet, whilst it has an IP

    1. make sure `echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf && sudo sysctl -p`
    2. and `sudo systemctl restart nftables.service`

## setup KVM disk

```bash
VM_NAME="archlinux"
VM_ISO="/data/ISOs/archlinux-2024.07.01-x86_64.iso"
# VM_URL="http://ftp.us.debian.org/debian/dists/stable/main/installer-amd64/"
VM_OS="archlinux"
VM_CORES=4
VM_RAMSIZE=4096
VM_IMG="/data/VMs/${VM_NAME}.qcow2"
VM_DISKSIZE=10
VM_NET="default"
VM_UEFI_FW="/usr/share/edk2/x64/OVMF_CODE.4m.fd"

qemu-img create -f qcow2 /data/VMs/${VM_NAME}.qcow2 10G
sudo chown libvirt-qemu:kvm /data/VMs/${VM_NAME}.qcow2

  sudo virt-install \
    --name "${VM_NAME}" \
    --cdrom "${VM_ISO}" \
    --os-variant="${VM_OS}" \
    --vcpus "${VM_CORES}" \
    --memory "${VM_RAMSIZE}" \
    --disk path="${VM_IMG}",size="${VM_DISKSIZE}",bus=virtio,format=qcow2 \
    --network network="${VM_NET}",model=virtio \
    --graphics vnc \
    --boot loader="${VM_UEFI_FW}",loader.readonly=yes,loader.type=pflash

# quick copy on write (qcow)
# virtual disk images don't actually take up 10G right on creation, but fill up gradually as you use and **cap at 10G**
```

**using virt-manager GUI**

1. select ISO
2. select the above created custom image, or create a custom from here
3. **customize configuration before install** and in network selection use NAT (as i've configured my nftables for NAT only)
4. then use UEFI in **config** in Overview > Hypervisor Details > Firmware > **UEFI x86_64: /usr/share/edk2/x64/OVMF_CODE.4m.fd** (4M firmware latest)

## Setup KVM network (nftables) config

### when using KVM in `NAT` network mode

essentially i need to allow myself to **route** the traffic b/w kvm and the router.

```bash linenums="1"
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

- allow `virbr0` to send me packets **(input)**
- allow me to **forward** the packets `virbr0` sends to me and send them back as well on the interface `wlp3s0` drop the rest
- and then allow the access to `virbr0` bridge to SNAT it's packets via me (the host) to the router.

```bash linenums="1"
flush ruleset

table ip FILTER_HOST {
        chain INPUT {
                ...
                iifname "virbr0" accept
                ...
        }

        chain FORWARD_KVM {
                type filter hook forward priority 0; policy drop;
                ct state established, related accept
                ct state invalid counter drop

                iifname "virbr0" oifname "wlp3s0" accept
                iifname "wlp3s0" oifname "virbr0" accept

                counter drop
        }
}


table ip NAT_KVM {
    chain POSTROUTING_KVM {
        type nat hook postrouting priority 100; policy accept;
        ip saddr 192.168.122.0/24 masquerade
    }
}
```

![image](https://i.ibb.co/nmtSDyr/A-13-21-20-01-Jul.png)

### creating a custom bridge

there are two types of bridges that can be created

## to backup VM

```bash linenums="1"
virsh list --all
virsh dumpxml <vm-name> > <vm-name>.xml
cp /data/VMs/archlinux.qcow2 /path/to/backup/<vm-disk>.qcow2
```

## to restore a VM

```bash linenums="1"
# first setup KVM properly

# then to restore
virsh define /path/to/backup/<vm-name>.xml
sudo chown libvirt-qemu:kvm /path/to/backup/<vm-disk>.qcow2
sudo chmod 660 /path/to/backup/<vm-disk>.qcow2
virsh start <vm-name>
virsh list --ALL
virt-viewer <vm-name>
```

## Install VirtIO Drivers for Windows Guests

VirtIO drivers are para-virtualized drivers for KVM guests. Unfortunately, Microsoft does not provide VirtIO drivers. If you want to create a Microsoft Windows virtual machine, you need to download the virtio-win.iso image, which contains the VirtIO drivers for the Windows operating system.

## virsh

virsh is a command line interface that can be used to create, destroy, stop start and edit virtual machines and configure the virtual environment (such as virtual networks etc)

```bash linenums="1"
virsh list --all

virsh start <vm-name>

virsh shutdown <vm-name>

virsh suspend <vm-name>

virsh resume <vm-name>

virsh reboot <vm-name>

virsh destroy <vm-name> # force stop

virsh undefine <vm-name> # delete
```

### network management

```bash linenums="1"
virsh net-list --all

virsh net-create <xml-file>

virsh net-destroy <net-name>

virsh net-start <net-name>

virsh net-destroy <net-name>
```

## virt-install

virt-install is a command line tool that simplifies the process of creating a virtual machine.

## virt-manager

virt-manager is a GUI that can be used to create, destroy, stop, start and edit virtual machines and configure the virtual environment (such as virtual networks etc).

- is the graphical user interface for libvirt
- that allows you to do all these virtualization things without ever touching a command line.
- Libvirt provides an abstracted api for storage, network, computer, and virtualization. so other programs or people can managed it by one interface instead of manually
- virt-manager can use libvirt and make it pretty for meatbags.
- there's's really no reason to work with bare qemu, it is nice to have Libvirt on top of it
- It can also use other virtualizers than QEMU, like Xen for example. It has an API, a scripting interface, a command line interface, a network connection interface, and...

## libvirt

is "glue" layer/library that arranges and manages QEMU sessions, its disks, the networks, and so on.

## QEMU

- QEMU = software emulation, KVM = Hardware emulation
- QEMU uses emulation; KVM uses processor extensions (HVM) for virtualization.
- QEMU is the virtualizer software that actually "runs" the virtual machines. It can run without KVM, but it's much much slower because then QEMU has to really "simulate" the guest CPU.
- KVM is type-1 and is a part of the linux kernel [link](https://cloudzy.com/wp-content/uploads/KVM-Vs-QEMU_Pic1.png)
- QEMU is a type-2 hypervisor (Quick Emulator QEMU)
- KVM is a feature of the kernel that Qemu can use to emulate. and pass through, the instructions to the actual processor of your computer, thereby improving performance by probably an order of magnitude.
- QEMU can be used on it's own but it's slow. KVM allows it access to the systems hardware.
- VirtualBOX can also use KVM for it's hardware components. but don't use virutalbox due to oracle garbage.
- QEMU is an CLI and userspace program to manage emulation and virtualization and can use KVM when it creates vitual machines.
- QEMU can use other hypervisors like Xen or KVM to use CPU extensions (HVM) for virtualization

## kvm vs wsl

- WSL is unique in its approach and doesn't fit neatly into the Type 1 or Type 2 hypervisor classifications
- WSL 2 introduces a full Linux kernel running in a lightweight VM. WSL 1 was just a compatibliity layer for running programs and had no virtualization.
- WSL 2 runs on top of Hyper-V architecture, which is a Type 1 hypervisor.
- Dynamic resource allocation with minimal overhead compared to traditional VMs.
- but Offers less isolation compared to KVM, KVM Provides strong isolation between VMs
- KVM: Near-native performance with hardware-assisted virtualization but with higher resource overhead. WSL
- You can think of WSL2 as a miniVM with features turned off and features striped away, but has better integration with Windows

## Hypervisors

hypervisor, also known as a Virtual Machine Monitor (VMM), creates and runs virtual machines. abstracts the physical hardware from the VMs and executes their procecesses

### Type 1 (Bare-Metal) Hypervisors

run directly on the host's hardware to control the hardware Microsoft Hyper-V, Xen and ### KVM + QEMU
. Xen is open source and is used by Amazon EC2. Better performance, efficiency, security and stability due to less layers b/w hardware. useful for Enterprise data centers, cloud service providers where performance and security is key

### Type 2 (Hosted) Hypervisors

rely on the host OS for device support and management VMware Workstation, VirtualBox, suitable for desktop and development environments

### core components of hypervisors

#### Hypervisor Kernel

The core component that directly interacts with the hardware, managing CPU, memory, and I/O resources.

#### Virtual Machine Manager

Manages the creation, execution, and deletion of VMs, handling resource allocation and scheduling

#### Device Emulation

Provides virtualized hardware devices to the guest VMs, including virtual disks, network interfaces, and graphic adapters.

#### VM Monitor (VMM)

Ensures the isolation and security of VMs, managing their execution state and context switching between VMs.

### how hypervisor worky?

- hypervisor allocates physical resources (CPU, memory, I/O) to each VM based on policies and configurations
- ensures that VMs do not interfere with each other
- schedules VM execution on physical CPU cores, managing context switches between VMs.
- uses memory overcommitment, ballooning, and page sharing, to optimize RAM usage
- ensure each VM has its own isolated memory space, preventing unauthorized access between VMs.
- emulation allows VMs to interact with virtual devices, which the hypervisor maps to the physical hardware.

<https://www.linux-kvm.org/page/FAQ#General_KVM_information>

### list of supported OSes in KVM:

<https://www.linux-kvm.org/page/FAQ#General_KVM_information>

## kvm vs xen

- Xen is an external hypervisor; it assumes control of the machine and divides resources among guests. On the other hand, KVM is part of Linux and uses the regular Linux scheduler and memory management
- KVM is much smaller and simpler
- KVM only run on processors that supports x86 hvm (vt/svm instructions set)
- Xen also allows running modified operating systems on non-hvm x86 processors using a technique called paravirtualization
- KVM does not support paravirtualization for CPU but may provide for device drivers

Go back [home](../).
