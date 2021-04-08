# Set bootlist in unattended network installation
Network installation of OCP is performed by setting the LPAR to boot from network either using DHCP or by pointing to a `bootp` server. The LPAR will then load a network boot image using `tftp` and such image will perform the OCP installation, usually exploiting a HTTP server.

The OCP installation is configured by providing "ignition" files that are created during the initial preparation.

The OCP default setup will not set the LPAR bootlist and at the end of the code installation the LPAR will start again the network boot at reboot.

You can manually open a console on the LPAR and enter SMS menu to change the bootlist or you can add a custom "ignition" file to the OCP process to set the bootlist, allowing installation to proceed unattended.

This page describes how to customize the OCP network installation.



## Create configuration file
Assuming you are installing OCP on `sda` disk, you need to request a post installation action that needs to be executed after code ahs been deployed.

Create a `boot-from-sda.fcc` file with the following content:

```text
variant: fcos
version: 1.1.0
storage:
  files:
    - path: /usr/local/bin/post-install-hook
      mode: 0755
      contents:
        inline: |
          #!/bin/bash
          bootlist -m normal -o sda
systemd:
  units:
    - name: post-install-hook.service
      enabled: true
      contents: |
        [Unit]
        Description=Run after install
        After=coreos-installer.service
        Before=coreos-installer.target
        [Service]
        Type=oneshot
        ExecStart=/usr/local/bin/post-install-hook
        [Install]
        RequiredBy=coreos-installer.target

```

This configuration file needs to be transformed in "ignition" format.


## Build `butane` tool
This tool can be downloaded from GitHub and compiled for ppc64le. It requires golang.

* `git clone https://github.com/coreos/butane.git`
* `yum install go`
* `cd butane`
* `./build`

The tool is compiled and made available in `bin/ppc64le/butane`.


## Create ignition file
Use `butane` as a filter to create the ignition file:

```text
butane < boot-from-sda.fcc > boot-from-sda.ign
```

## Add ignition file in network boot configuration
Network boot requires the new "ignition" file to be made available through the selected protocol, which is usually HTTP. The file `boot-from-sda.ign` should be copied on the web server's document directory.

The network boot process is configured by the `grub.cfg` file that should be updated like the following snippet:


```text
menuentry "Bootstrap CoreOS (BIOS)" {
echo "Loading kernel Bootstrap"
linux "/rhcos-4.7.0-ppc64le-live-kernel-ppc64le" rd.neednet=1 ip=172.17.227.9::172.17.224.1:255.255.224.0:tfv-ocp-bastion.power.seg.it.ibm.com:env32:none nameserver=172.17.224.5 nameserver=172.17.224.6 coreos.inst=yes coreos.inst.install_dev=sda coreos.live.rootfs_url=http://172.17.227.23:80/ocp47/rhcos-4.7.0-ppc64le-live-rootfs.ppc64le.img coreos.inst.ignition_url=http://172.17.227.23:80/bootstrap.ign ignition.config.url=http://172.17.227.23:80/boot-from-sda.ign ignition.firstboot ignition.platform.id=metal
echo "Loading initrd"
initrd "/rhcos-4.7.0-ppc64le-live-initramfs.ppc64le.img"
}
```

The added parameters are `ignition.config.url=<boot-from-sda.ign URL> ignition.firstboot ignition.platform.id=metal`. They are all needed.
