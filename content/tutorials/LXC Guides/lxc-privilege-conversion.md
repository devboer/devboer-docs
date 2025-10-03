---
title: "Lxc Privilege Conversion"
date: 2025-09-27T20:19:13-05:00
draft: false
---
**Goal** Align privileged LXC Jellyfin container's permissions with unprivileged Servarr companion services.

**Catalyst** Since, I have introduced Servarr, I was having wonky issues with Jellyfin losing controls over its media. And, I learned privileged containers means same IDs as host, I had to remedy this pronto!

So far, I have learned, PVE 9 doesn’t have un|shift pct command nor uidmappingshift (something like this) to convert LXC container from privileged to unprivileged.

I imagined this going smoother when using ChatGPT and web searches. I had a few challenges with my Jellyfin LXC being privileged and my servarr instances with unprivileged access.
- Inside privileged container it had two files to remove 'random' and 'urandom' that were device types inside /var. (Correction?)
- Comment out specific lines in my container config file: lxc.idmap and cgroup access to GPU
- Backup container with the container 'cleaned' of the above
- Restore with unprivileged switch to 1

As I recall the challenges above were resolved when trying to restore the container. Each time I failed I restored it to privileged and fixed the requirements until I could restore it unprivileged.

### Without any challenges, I was ready to complete the project...
1. `pct stop <container-id>`

2. `vzdump <container-id> —dumpdir /path/to/backup`

3. `pct destroy <container-id>` or last for backup but need a different container id for restore

4. A) `pvesm status` to see my storage options

4. B) `pct restore <new-container-id> /path/to/backup —storage <storage_pool> —unprivileged 1`

From here I had problems with jellyfin’s directories owned by ‘nobody’ and group ‘adm’ UID: 65534; GID: 4. And, this was the issue before I stumbled upon container conversion. Just now it is only on Jellyfin's directories, funny enough.

So lots of trying things with help of searching the web and using ChatGPT and DeepSeek. I noticed nothing works, dead end reached inside container operations with “not permitted” errors.

And so, I learned something about Proxmox, where it keeps its containers when mounted.

`pct stop <container-id>`
[!info] https://etldispatch.com/blog/reviving-a-broken-lxc-container/

[!info] https://cloudspinx.com/convert-unprivileged-container-in-proxmox-to-privileged/

`pct mount <container-id>`

chown -R 100000:100000 /var/lib/lxc/<ct-id>/rootfs/path/to/change

The six-digit UID:GID is the ID of root inside container that will convert to UID:GID of ‘0’.

Problem solved!

From here, I started container back up and “chowned” the directory to ‘jellyfin’

- JELLYFIN_DATA_DIR="/var/lib/jellyfin"
- JELLYFIN_CONFIG_DIR="/etc/jellyfin"
- JELLYFIN_LOG_DIR="/var/log/jellyfin"
- JELLYFIN_CACHE_DIR="/var/cache/jellyfin"

Nvidia device files are also owned by ‘nobody’. After much delibration and trial + error, I find this to not matter.

I have an unprivileged container with proper uid:gid mapping from host to container and a working GPU binding. The GPU issue is something I'll report on later when I learn more about what kernel issues I had with nvidia installer. This time around Nvidia installer ran without issue on the host like before. 

### Connection complete, and ON to the next adventure!

## Next Up ...
*I'd like to learn how to run my LXC container's service rootless similar to Docker*