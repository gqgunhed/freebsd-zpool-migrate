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

# Preparation
- Install a new FreeBSD 11.1 system with root-on-ZFS
  - check and copy new options from:
    - /etc/sysctl.conf
    - /boot/loader.conf
    - zfs get (all options (checksum, noatim, compress, etc.) from pool ZFS datasets)
    - GPT partition layout

# Procedure
- rpool: existing/productive zpool
- newpool: newly created zpool

## Variant a) local transfer
- Install the two new drives into existing system
- Prepare the new disks (layout) as seen from step "Preparation"
- `zpool create mirror newpool ada0 ada1` <-- needs to be corrected, pay special attention to the correct device names!
- make snapshot of existing system
  - `zfs snapshot -r rpool@migrate-start`
- transfer data to new zpool
  - `zfs send -R rpool@migrate-start | zfs receive newpool`
- Install bootcode onto new drives
- reboot to live FreeBSD 11.1 system from USB-stick
- rename zpool with `zpool import newpool rpool`
- `zpool export rpool`
- reboot the system, detach old drives
### Upgrade to new FreeBSD version
- follow the instructions from https://www.freebsd.org/doc/en/books/handbook/updating-upgrading-freebsdupdate.html

## Variant b) remote transfer
- make snapshot of existing system
  - `zfs snapshot -r rpool@migrate-start`
- temporarily allow root SSH access on target system
- transfer over to new/target system
  - `zfs send -R rpool@migrate-start | ssh root@targethost "zfs receive -Fdu rpool"`
### optional (test)
exclude the / directory from the transfer and use a sane starting point
```
zfs snapshot -r rpool@migrate-start
zfs list -H -o name rpool > /zfs.txt
# remove any unwanted ZFS datasets from zfs.txt
for i in "`cat /zfs.txt`"; do zfs send -R ${i} | ssh root@targethost "zfs receive -Fdu rpool"
# okay, that includes the snapshots, BUT also the child datasets ...
```


### after transfer on target system
- remove root SSH access
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
