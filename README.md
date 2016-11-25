#Breakdown

After a few days (years) of web browsing I still cannot understand why there is no [pygrub](http://wiki.xen.org/wiki/PyGrub) alternative for qemu-kvm/libvirt. There are some workarounds around there, but all solutions have drawbacks (simplified objections):

1. Move your root filesytem to a disk with a parition table and install grub.  
 ⬇ Too tangled to create/resize filesystem, a common task with VMs.
2. Create a small disk with /boot partition and grub[[Gadi's Blog](http://blog.gadi.cc/better-lvm-for-kvm/)].  
 ⬇ Requires to waste time (config, setup) and space (extra disk).
3. Use [PvGrub](https://wiki.debian.org/PvGrub).  
 ⬇ I think this is an overhead for VMs (config) and I like to set kernel args in the domain configuration. I also known about this too late.
4. [Use kexec hackery](https://wiki.archlinux.org/index.php/QEMU#By_specifying_kernel_and_initrd_manually)  
 ⬇ It's not the best option for most people.
5. Surrender. Use host kernel or a selection of kernels stored some where.  
 ⬇ Requires initrd with most modules, but some times, the module (like iptables modules) that you need is not there... regenerate initrd, reboot... I don't like it.
 
[Pygrub](http://wiki.xen.org/wiki/PyGrub) is intended to be used with xen-tool to find /boot/grub/menu.lst and locate the kernel/initrd pair to be loaded. Also, in debian, exists the pv-grub-menu package which creates the /boot/grub/menu.lst. Clean and easy.

Libvirt allows the domain/os/bootloader[[1](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Virtualization_Administration_Guide/sub-sect-op-sys-host-boot.html)][[2](https://libvirt.org/formatdomain.html#elementsOSBootloader)] parameter to be set to pygrub but just works for Xen domains. I'd like to deepen into libvirt and allow kvm domains to use pygrub. However, I do not have enough time and the following workaround seems to be really functional.

#Solution
The qemu script is a hook for libvirt. Works as a wrapper for pygrup:

+ Parses the domain configuration on prepare/begin event,
+ lets pygrub find the kernel through the available disks and
+ copy the images to the REF_PATH/DOMAIN folder

#Installation (Based on Debian host/guest)

+ Host (Dom0)
    + Copy qemu into /etc/libvirt/hooks/ and chmod 755
    + Install xen-utils-X.X to get pygrub (apt-get install --no-install-recommends xen-utils-4.1)
    + Install xmlstarlet and coreutils (basename/dirname) (apt-get install xmlstarlet coreutils)
    + Customize the script: set pygrub path (PYGRUB) and kernel images storage base path (BASE_PATH)
    + Restart libvirtd (service libvirt-bin restart or the equivalent for systemd)

+ Guest (DomU)
    + Install linux-image-amd64 (or the one required for your VM)
    + If grub was installed, purge all grub packages.
    + Configure /boot/gurb/menu.lst
        + create a simple menu.lst with just "title, kernel and initrd" pointing to /vmlinuz /inird.img
        + or install pv-grub-menu, remove menu.lst and run update-menu-lst
    + If your are moving Xen VM to KVM review fstab, interfaces and persistent-net rules.
    + Configure Domain (note that DOMAIN is the exact libvirt domain name):
        + domain/os/kernel set to REF_PATH/**DOMAIN**/vmlinuz, ex: /srv/kvm/kernel-guest/**test-vm**/vmlinuz
        + domain/os/initrd set to REF_PATH/**DOMAIN**/initrd, ex: /srv/kvm/kernel-guest/**test-vm**/initrd
        + domain/os/cmdline the kernel command line. The command line set in menu.lst is ignored.
