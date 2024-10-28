# Ansible Role: NetBoot

This role provides network boot functionality for both legacy BIOS and UEFI systems.

The PXELinux, GRUB2 and iPXE boot environments are accessible over both TFTP and HTTP.

Next to the boot environments, also ISO images can be made available over HTTP.

Optionally a DHCP4/6 server can be configured to provide the DHCP Netboot options for a subnet.

## Requirements

This role has to be executed as user 'root'.

This role should be installed on an UEFI based system to enable UEFI based network booting.

## Tags

Following tags are supported by this role:

  * dhcp
  * tftp
  * nginx
  * autofs
  * pxelinux
  * grub2
  * ipxe

## Variables

### TFTP

A TFTP server provides tftp access to the available bootloaders.

| Variable          | Type   | Required | Default             | Comment             |
|-------------------|--------|----------|---------------------|---------------------|
| netboot_tftp_root | string | No       | '/var/lib/tftpboot' | TFTP root directory |

### Nginx

The Nginx webserver provides web access (HTTP) to the available bootloaders and ISO images.

| Variable           | Type   | Required | Default | Comment                   |
|--------------------|--------|----------|---------|---------------------------|
| netboot_webroot    | string | No       | '/www'  | Web root directory        |
| netboot_nginx_ipv4 | string | No       | '*'     | Nginx IPv4 listen address |
| netboot_nginx_ipv6 | string | No       | '[::]'  | Nginx IPv6 listen address |
| netboot_nginx_port | string | No       | 80      | Nginx listen port         |

### AutoFS

AutoFS is used to provide dynamic web access towards the configured ISO images.

These images will be reachable via: http://<NETBOOT_SERVER>/autofs/<NAME>

| Variable            | Type   | Required | Default | Comment            |
|---------------------|--------|----------|---------|--------------------|
| netboot_autofs_isos | list() | No       | N/A     | AutoFS ISO images  |

```yaml
# ISOs to dynamically load when accessed
netboot_autofs_isos:
  # URL: http://<NETBOOT_SERVER>/autofs/alma9-minimal
  - name: alma9-minimal
    iso: /isos/AlmaLinux-9-latest-x86_64-minimal.iso
  - name: alma9-live
    iso: /isos/AlmaLinux-9-latest-x86_64-Live-GNOME-Mini.iso
```

### PXELinux

PXELinux is a Syslinux based bootloader which can be used to PXE boot in a BIOS based system.

The PXELinux files are reachable via:

  * tftp://<NETBOOT-SERVER>/pxelinux

| Variable                | Type   | Required | Default   | Comment                                                          |
|-------------------------|--------|----------|-----------|------------------------------------------------------------------|
| netboot_pxelinux_files  | list() | No       | See below | Local PXELinux bootloader files to copy to TFTP/HTTP directories |
| netboot_pxelinux_labels | dict() | No       | N/A       | PXELinux bootloader labels                                       |

```yaml
# Default PXELinux files, copied from 'syslinux-tftpboot' package files
netboot_pxelinux_files:
  - /tftpboot/pxelinux.0
  - /tftpboot/ldlinux.c32
  - /tftpboot/libutil.c32
  - /tftpboot/menu.c32
  - /tftpboot/memdisk

# PXELinux label configuration
netboot_pxelinux_labels:
  local: |
    menu label Boot from ^local drive
    menu default
    localboot 0xffff
  rescue: |
    menu label ^Rescue installed system
    kernel http://netboot/alma9-minimal/images/pxeboot/vmlinuz
    append initrd=http://netboot/alma9-minimal/images/pxeboot/initrd.img inst.rescue inst.repo=http://netboot/alma9-minimal/
  rescue_ssh: |
    menu label Rescue installed system (SSH ENABLED)
    text help
          WARNING: This will provide passwordless root access!
    endtext
    kernel http://netboot/alma9-minimal/images/pxeboot/vmlinuz
    append initrd=http://netboot/alma9-minimal/images/pxeboot/initrd.img inst.rescue inst.sshd inst.repo=http://netboot/alma9-minimal/
  install: |
    menu label ^Install AlmaLinux 9.4
    kernel http://netboot/alma9-minimal/images/pxeboot/vmlinuz
    append initrd=http://netboot/alma9-minimal/images/pxeboot/initrd.img inst.repo=http://netboot/alma9-minimal/
  live: |
    # Options to boot the live image can be found in 'man dracut.cmdline -> Booting live images'
    menu label ^Start AlmaLinux Live 9.4
    kernel http://netboot/alma9-live/images/pxeboot/vmlinuz
    append initrd=http://netboot/alma9-live/images/pxeboot/initrd.img root=live:http://netboot/alma9-live/LiveOS/squashfs.img rd.live.image
  memtest: |
    menu label Run a ^memory test
    kernel http://netboot/alma9-minimal/isolinux/memtest
```

### GRUB2

The GRUB2 bootloader can be used to PXE or HTTP boot an UEFI based system.

The GRUB2 files are reachable via:

  * tftp://<NETBOOT-SERVER>/grub2
  * http://<NETBOOT-SERVER>/grub2

| Variable              | Type   | Required | Default   | Comment                                                       |
|-----------------------|--------|----------|-----------|---------------------------------------------------------------|
| netboot_grub2_files   | list() | No       | See below | Local GRUB2 bootloader files to copy to TFTP/HTTP directories |
| netboot_grub2_entries | list() | No       | N/A       | GRUB2 bootloader menu entries                                 |

```yaml
# Default GRUB2 EFI files
netboot_grub2_files:
  - /boot/efi/EFI/almalinux/shim.efi    # Use this pre-bootloader when secure boot is enabled
  - /boot/efi/EFI/almalinux/grubx64.efi

# GRUB2 Menu entries configuration
netboot_grub2_entries:
  - title: Local boot
    entry: |
      exit
  - title:  Rescue installed system
    entry: |
      echo 'Loading Linux ...'
      linuxefi (http,netboot)/alma9-minimal/images/pxeboot/vmlinuz inst.rescue inst.repo=http://netboot/alma9-minimal/
      echo 'Loading initial ramdisk ...'
      initrdefi (http,netboot)/alma9-minimal/images/pxeboot/initrd.img
  - title:  Rescue installed system (SSH ENABLED)
    entry: |
      echo 'Loading Linux ...'
      linuxefi (http,netboot)/alma9-minimal/images/pxeboot/vmlinuz inst.rescue inst.sshd inst.repo=http://netboot/alma9-minimal/
      echo 'Loading initial ramdisk ...'
      initrdefi (http,netboot)/alma9-minimal/images/pxeboot/initrd.img
  - title: Install AlmaLinux 9.4
    entry: |
      echo 'Loading Linux ...'
      linuxefi (http,netboot)/alma9-minimal/images/pxeboot/vmlinuz inst.rescue inst.repo=http://netboot/alma9-minimal/
      echo 'Loading initial ramdisk ...'
      initrdefi (http,netboot)/alma9-minimal/images/pxeboot/initrd.img
  - title: Start AlmaLinux Live 9.4
    entry: |
      echo 'Loading Linux ...'
      linuxefi (http,netboot)/alma9-live/images/pxeboot/vmlinuz root=live:http://netboot/alma9-live/LiveOS/squashfs.img rd.live.image
      echo 'Loading initial ramdisk ...'
      initrdefi (http,netboot)/alma9-live/images/pxeboot/initrd.img
  - title: UEFI Firmware Settings
    entry: |
      fwsetup
```

**Note:** Shim does not handle DHCP option 67 well, it ignores the path when it loads other files (grubx64.efi, ...).
          Only the DHCP header 'boot file' will work. https://bugzilla.redhat.com/show_bug.cgi?id=1777244

### iPXE

Both the iPXE ROMs and an iPXE boot script are provided, which provide bootloader functionality for both
BIOS and UEFI based systems.

The GRUB2 files are reachable via:

  * tftp://<NETBOOT-SERVER>/ipxe
  * http://<NETBOOT-SERVER>/ipxe
  * tftp://<NETBOOT-SERVER>/ipxe/scripts/boot.ipxe
  * http://<NETBOOT-SERVER>/ipxe/scripts/boot.ipxe

**Note:** iPXE does not support UEFI Secure Boot by default. The ROMs are not signed.

| Variable           | Type   | Required | Default   | Comment                                           |
|--------------------|--------|----------|-----------|---------------------------------------------------|
| netboot_ipxe_files | list() | No       | See below | Local iPXE files to copy to TFTP/HTTP directories |
| netboot_ipxe_items | list() | No       | N/A       | iPXE boot script items                            |

```yaml
# Default iPXE ROM files, provided by the 'ipxe-bootimgs-x86' package
netboot_ipxe_files:
  - /usr/share/ipxe/undionly.kpxe
  - /usr/share/ipxe/ipxe.lkrn
  - /usr/share/ipxe/ipxe-x86_64.efi
  - /usr/share/ipxe/ipxe-snponly-x86_64.efi

# iPXE boot script items
netboot_ipxe_items:
  - item: rescue
    title: Rescue installed system
    key: r
    script: |
      set base http://netboot/alma9-minimal
      echo Loading Linux ...
      kernel ${base}/images/pxeboot/vmlinuz initrd=initrd.img inst.rescue inst.repo=${base}
      echo Loading initial ramdisk ...
      initrd ${base}/images/pxeboot/initrd.img
      boot || goto failed
      goto start
  - item: rescue_ssh
    title: Rescue installed system (SSH ENABLED)
    script: |
      set base http://netboot/alma9-minimal
      echo Loading Linux ...
      kernel ${base}/images/pxeboot/vmlinuz initrd=initrd.img inst.rescue inst.sshd inst.repo=${base}
      echo Loading initial ramdisk ...
      initrd ${base}/images/pxeboot/initrd.img
      boot || goto failed
      goto start
  - item: install
    title: Install AlmaLinux 9.4
    key: i
    script: |
      set base http://netboot/alma9-minimal
      echo Loading Linux ...
      kernel ${base}/images/pxeboot/vmlinuz initrd=initrd.img inst.repo=${base}
      echo Loading initial ramdisk ...
      initrd ${base}/images/pxeboot/initrd.img
      boot || goto failed
      goto start
  - item: live
    title: Start AlmaLinux Live 9.4
    key: l
    script: |
      set base http://netboot/alma9-live
      echo Loading Linux ...
      kernel ${base}/images/pxeboot/vmlinuz initrd=initrd.img root=live:${base}/LiveOS/squashfs.img rd.live.image
      echo Loading initial ramdisk ...
      initrd ${base}/images/pxeboot/initrd.img
      boot || goto failed
      goto start
```

### DHCP

Optionally, the netboot server can also provide DHCP server functionality, including the netboot parameters.

| Variable                    | Type   | Required | Default | Comment                                                               |
|-----------------------------|--------|----------|---------|-----------------------------------------------------------------------|
| netboot_dhcp_authoritative  | bool   | No       | false   | THis DHCP server is the official DHCP server for the local networks   |
| netboot_dhcp6_authoritative | bool   | No       | false   | This DHCP6 server is the official DHCP6 server for the local networks |
| netboot_dhcp_subnets        | list() | No       | N/A     | DHCP subnet configurations                                            |
| netboot_dhcp6_subnets       | list() | No       | N/A     | DHCP6 subnet configurations                                           |

```yaml
# DHCP IPv4 configuration
netboot_dhcp_subnets:
  - subnet: 192.168.2.0
    netmask: 255.255.255.0
    range: 192.168.2.200 192.168.2.220
    options:
      routers: 192.168.2.1
      domain-name-servers: 192.168.2.1
    classes:
      ipxe: |
        match if substring (option user-class, 0, 4) = "iPXE";
        filename "http://netboot/ipxe/scripts/boot.ipxe";
      pxeclients: |
        match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
        # TFTP Server
        next-server 192.168.2.10;
        
        # EFI X86-64
        if option architecture-type = 00:07 {
          # GRUB2 bootloader (secure boot support)
          filename "grub2/shim.efi";
          # Chainload iPXE (no secure boot support!)
          #filename "ipxe/ipxe-snponly-x86_64.efi";
        }
        
        # Legacy BIOS
        if option architecture-type = 00:00 {
          # PXELinux bootloader
          filename "pxelinux/pxelinux.0";
          # Chainload iPXE
          #filename "ipxe/undionly.kpxe";
        }
      httpclients: |
        match if substring (option vendor-class-identifier, 0, 10) = "HTTPClient";
        option vendor-class-identifier "HTTPClient";

        # GRUB2 bootloader (secure boot support)
        filename "http://netboot/grub2/shim.efi";
        # Chainload iPXE (no secure boot support!)
        #filename "ipxe/ipxe-snponly-x86_64.efi";

# DHCP IPv6 configuration
netboot_dhcp6_subnets:
  - subnet6: fd4e:9193:d923:2::/64
    range6: fd4e:9193:d923:2::200 fd4e:9193:d923:2::220
    options:
      dhcp6.name-servers: fd4e:9193:d923:2::1
    classes:
      pxeclients: |
        match if substring (option dhcp6.vendor-class, 6, 9) = "PXEClient";
        option dhcp6.bootfile-url "tftp://[fd4e:9193:d923:2::1:2]/grub2/shim.efi";
      httpclients: |
        match if substring (option dhcp6.vendor-class, 6, 10) = "HTTPClient";
        option dhcp6.bootfile-url "http://[fd4e:9193:d923:2::1:2]/grub2/shim.efi";
        option dhcp6.vendor-class 0 10 "HTTPClient";
```
