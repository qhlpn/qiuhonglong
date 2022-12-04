https://serverfault.com/questions/849966/zfs-delete-snapshots-with-interdependencies-and-clones

https://stackoverflow.com/questions/66925115/zfs-filesystem-has-dependent-clones







```
zfs list -t all -o name,origin,clones
zfs create zfs-pool/test
zfs snapshot zfs-pool/test@1
zfs rollback zfs-pool/test@1

zfs clone zfs-pool/test@1 zfs-pool/test2
zfs send zfs-pool/test@1 | pv | zfs receive zfs-pool/test3
```