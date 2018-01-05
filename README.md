# freebsd-zpool-migrate
*Disclaimer: provided as-is, so no guarantee. Use this information at your own risk!*

Small script how to transfer my existing ZFS pool to another disk layout.


# Starting Point
Single steps to move 
- from: 
  - FreeBSD 10.3-RELEASE
  - 2x 2TB WD RED
  - 2x 3TB WD RED
- to:
  - FreeBSD 11.1-RELEASE
  - 2x 8TB WD RED

# Initialization
- Install a new FreeBSD 11.1 system with root-on-ZFS
  - check and copy new options from:
    - /etc/sysctl.conf
    - /boot/loader.conf
    - zfs get (all options (checksum, noatim, compress, etc.) from pool ZFS datasets)
    - GPT partition layout
  - new pool: `rpool`, same as old zpool
  - activate mirror and full disk encryption via installer
  
## Preparation
- install necessary tools: `pkg install tmux vim-lite`

## Remote transfer
### source system
- make snapshot of existing system
  - `zfs snapshot -r rpool@migrate-start`

### target (new) system
- temporarily allow root SSH access on target system
  - `vim /etc/ssh/sshd_config` and activate/change `PermitRootLogin yes`
  - restart sshd with `service sshd restart`
  - `zfs create rpool/zz_MIGRATE` create new target ZFS dataset to temporarily hold the transferred datasets

### source system
- transfer over to new/target system
  - `zfs send -R rpool@migrate-start | ssh root@targethost "zfs receive -Fdu rpool"`

```
zfs list -H -o name -d 1 rpool | grep "rpool/" > /zfs.txt # skip root filesystem
#for i in "`cat /zfs.txt`"; do zfs send -R ${i} | ssh root@targethost "zfs receive -Fdu rpool"
for set in $(zfs list -H -d 1 -o name | grep "rpool/"); do zfs send -DPRv $set@migration-start | ssh  10.16.0.1 "zfs recv -dFu rpool/zzz_MIGRATE"; done;
# zfs send: D-deduplicate, Pv-print progress, R-recursive
# zfs recv: d-drop pool name, F-force rollback to last snap, u-do NOT mount
```


### after transfer on target system
- remove root SSH access
- move ZFS datasets to correct(previous) location
- copy old configs and stuff (e.g. `/root`)
- add SSD cache/ZIL devices again


# References
- see my own zfs-backup script
## FreeBSD handbook links
- https://www.freebsd.org/doc/en/books/handbook/updating-upgrading-freebsdupdate.html
- https://www.freebsd.org/doc/en/books/handbook/zfs-zpool.html
- https://www.freebsd.org/doc/en/books/handbook/zfs-zfs.html
## man pages
- https://www.unix.com/man-page/freebsd/8/zpool/
- https://www.unix.com/man-page/freebsd/8/zfs/
## other links
- https://unix.stackexchange.com/questions/263677/how-to-one-way-mirror-an-entire-zfs-pool-to-another-zfs-pool
- https://forums.freenas.org/index.php?threads/howto-migrate-data-from-one-pool-to-a-bigger-pool.40519/
