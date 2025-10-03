---
title: "GPU passthrough to LXC on PVE"
date: 2024-08-28T01:01:28-05:00
draft: false
---

# Bind GPU to Unprivileged LXC   
### Tutorials Used:   
- [TheOrangeOne.net](https://theorangeone.net/posts/lxc-nvidia-gpu-passthrough/)   
- [YouTube Video using above tutorial](https://www.youtube.com/watch?v=-Us8KPOhOCY)   
- [Mount Storage](https://virtualizeeverything.com/2022/05/18/passing-usb-storage-drive-to-proxmox-lxc/)   
- [GPU Matrix](https://developer.nvidia.com/video-encode-and-decode-gpu-support-matrix-new) for Decode/Encode Reference   
- Greg suggesting libnvidia-encode/decode on guestOS and learned to use -### corresponding with driver version   
   
# Proxmox Setup   
- Update Proxmox & Reboot   
- Install pve-headers & build-essentials   
- update-initramfs   
- blacklist free nvidia driver   
- download official nvidia driver   
- chmod +x driver   
- ./run install   
- install 32-bit also   
- ignore other prompts   
- edit modules.d files for loading and conf   
- create a rules file   
- reboot   
- verify driver is working with `nvidia-smi`   
- also could try `lshw -C display` this will list driver it should say 'nvidia'   
- Create CT with ubuntu compatible with jellyfin (18, 20, 22 not 22.10 work)   
- Do not start CT on creation   
- Edit CT conf file with   
   
```
# Allow cgroup access
lxc.cgroup.devices.allow: c 195:* rwm
lxc.cgroup.devices.allow: c 243:* rwm

# Pass through device files
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file

```
> [!Note]   
> =='cgroup access'== part corresponds with the character files below in =='device files'==   
> "195:\*" is using a wildcard for the device files. You can also write in this number too.   
> Also you can write in more IDs if you notice it changing on reboots to make certain you don't lose passthrough access.   
> Find these IDs using ls -l /dev/nv* and ignore 'caps' files.   

```
arch: amd64
cores: 2
hostname: MediaSrvUbuntu
memory: 4096
# mp0 mounts your drive to container (there is a process to mount it into PVE)
# mp#: /path/to/drive/dir,mp=/your/custom/path/on_ct
mp0: /mnt/data/multimedia,mp=/home/admin/data
net0: name=eth0,bridge=vmbr0,gw=192.168.50.1,hwaddr=32:07:27:92:00:A7,ip=192.168.50.20/24,tag=50,type=veth
ostype: ubuntu
rootfs: local-lvm:vm-121-disk-0,size=12G
swap: 512
# Group Access to Device Files
# lxc.cgroup2.devices.allow: c ID1:ID2 rwm
# rw gives reading and writing and m, I don't know
lxc.cgroup2.devices.allow: c 195:0 rwm
lxc.cgroup2.devices.allow: c 195:255 rwm
lxc.cgroup2.devices.allow: c 510:0 rwm
lxc.cgroup2.devices.allow: c 195:254 rwm
lxc.cgroup2.devices.allow: c 510:1 rwm
lxc.cgroup2.devices.allow: c 10:144 rwm
# Character/Device Files matching above IDs
# Find them with 'ls -l /dev/nv*' first column and second column after user owner
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
lxc.mount.entry: /dev/nvram /dev/nvram none bind,optional,create=file

```
- Boot Container   
- Update & Reboot   
- Install nvidia drivers with argument `--no-kernel-module` as container will use host's kernel modules   
- Again ignore messages and install 32-bit compatibilities   
- Install `nvtop`, `libnvidia-encode-525` (this will include -decode) ⟵ This needs to match Driver version   
- `update initramfs -u`   
- reboot   
- Now container and hypervisor are in sync on driver versions.   
- Test (verify) `nvidia-smi`   
   
> [!Note]   
> You can only see processes running from host with command nvidia-smi while on the guest you can use nvtop   

These were the steps I used.... to ready container for hardware transcoding.   
# Jellyfin installation   
Steps followed from here [here](https://jellyfin.org/docs/general/installation/linux/#ubuntu-repository)   
You can use the script or do the scripts instructions by hand from the steps below script.
`curl https://repo.jellyfin.org/install-debuntu.sh \| sudo bash`
or
`wget -O- https://repo.jellyfin.org/install-debuntu.sh \| sudo bash`   
I edited out 'sudo' as I was doing this from root.   
   
Some LXC packages that nvidia driver installer suggested…. libglvnd-dev, pkg-config, and libvulkan1   
Host modifications are missing in this guide or rather details. I'm sure they are in the links above but… why   
   
As of Mar 31, 2024…. they still match config and on both guest and host have same driver version but I have an error on 'nvidia-smi' command on guest side. Reboot?   
crw-rw-rw-  195,254 root root  31 Mar 02:31  /dev/nvidia-modeset
crw-rw-rw-    237,0 root root  31 Mar 02:31  /dev/nvidia-uvm
crw-rw-rw-    237,1 root root  31 Mar 02:31  /dev/nvidia-uvm-tools
crw-rw-rw-    195,0 root root  31 Mar 02:31  /dev/nvidia0
crw-rw-rw-  195,255 root root  31 Mar 02:31  /dev/nvidiactl
crw———-   10,144 root root  27 Mar 19:14  /dev/nvram   
/dev/nvidia-caps:
Permissions  Size User Group Date Modified Name
cr————  240,1 root root  31 Mar 02:31  nvidia-cap1
cr—r—r—  240,2 root root  31 Mar 02:31  nvidia-cap2   
   
Decoding Matrix   
|                BOARD | FAMILY | NVDEC Generation | Desktop/Mobile | Number of Chips | Total # of NVDEC | MPEG-1 | MPEG-2 | VC-1 | VP8 | VP9 8 Bit | VP9 10 Bit | VP9 12 Bit | H.264(AVCHD) | H.265 (HEVC) 4:2:0  8 Bit | H.265 (HEVC) 4:2:0 10 Bit | H.265 (HEVC) 4:2:0 12 Bit | H.265 (HEVC) 4:4:4 8 Bit | H.265 (HEVC) 4:4:4 10 Bit | H.265 (HEVC) 4:4:4 12 Bit | AV1 8 Bit | AV1 10 Bit |
|:---------------------|:-------|:-----------------|:---------------|:----------------|:-----------------|:-------|:-------|:-----|:----|:----------|:-----------|:-----------|:-------------|:--------------------------|:--------------------------|:--------------------------|:-------------------------|:--------------------------|:--------------------------|:----------|:-----------|
| Quadro P2000 / P2200 | Pascal |          3rd Gen |              D |               1 |                1 |    YES |    YES |  YES |  NO |       YES |        YES |        YES |          YES |                       YES |                       YES |                       YES |                       NO |                        NO |                        NO |        NO |         NO |

Translation on Jellyfin Playback NVIDIA NVENC Decoding checks   
[ x ] H264   
[ x ] HEVC   
[ x ] MPEG2   
[ ? ] MPEG4   
[ x ] VC1   
[ - ] VP8   
[ x ] VP9   
[ - ] AV1   
[ ? ] HEVC 10bit   
[ x ] VP9 10bit   
   
