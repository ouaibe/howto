
# ZFS : Migrating a 2-Drive RaidZ-1 to a Mirror Configuration

#### This is a risky operation that trusts ZFS's recovery feature, it is not production-ready and should only be done as a last resort.

If you mistakenly created a **raidz** pool instead of a **mirror** pool and then realised there is no benefit from 2-drive raidz configuration, here are the steps you can follow in order to break the **raidz** config and transform it into a **mirror** pool, hopefully *without losing data* (no guarantee there, have backups, I'm not responsible for your data losses, etc.).

1) First, query the status of your **zpool**

	
```
$ zpool status
```
```
pool: data
state: ONLINE
scan: none requested
config:

	NAME              STATE     READ WRITE CKSUM
	data              ONLINE       0     0     0
	  raidz1-0        ONLINE       0     0     0
	   disk1          ONLINE       0     0     0
	   disk2          ONLINE       0     0     0
```

And your ZFS datasets :

```
$ zfs list
```
```
data        1.28T  1.40T    96K  none
data/data   1.28T  1.40T  1.28T  /data
```

2) Then you need to chose one of the **vdevs** and put it offline, for instance, the second one :

```
$ zpool offline data disk2
```

This will effectively disable the vdev :

```
$ zpool status
```

```
pool: data
state: DEGRADED
status: One or more devices has been taken offline by the administrator.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using 'zpool online' or replace the device with
        'zpool replace'.
scan: none requested
config:

	NAME              STATE     READ WRITE CKSUM
	data              DEGRADED     0     0     0
	  raidz1-0        DEGRADED     0     0     0
	    disk1         ONLINE       0     0     0
	    disk2         OFFLINE      0     0     0


errors: No known data errors
```

3) Then comes the tricky part, you need to destroy the current pool :

```
$ zpool destroy data
```

4) Then create a new pool using the disk you did put offline, using its full device path (here I'm using a /dev/mapper/disk2 but replace this with your own):

```
$ zpool create -O mountpoint=none -o ashift=12 -f data2 /dev/mapper/disk2
```

5) When this is done, eventhough you destroyed the original pool, data is still present on the first **vdev**, namely *disk1*.

We will now re-import this vdev as the original pool : if this steps fails, you're toast.

```
$ zpool import -D data
```

There are other parameters available for `zpool import`, make sure to read the man pages.

6) When/if this has worked, you should now have two distinct pools, the old one in degraded mode, and the new one online with a single **vdev**.

```
$ zpool list
```
```
NAME    SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
data   5.44T  2.56T  2.88T         -    25%    47%  1.00x  DEGRADED  -
data2  2.72T   324K  2.72T         -     0%     0%  1.00x  ONLINE  -
```

And :

```
$ zpool status
```
```
pool: data
state: DEGRADED
status: One or more devices has been taken offline by the administrator.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using 'zpool online' or replace the device with
        'zpool replace'.
scan: none requested
config:

	NAME              STATE     READ WRITE CKSUM
	data              DEGRADED     0     0     0
	  raidz1-0        DEGRADED     0     0     0
	    disk1         ONLINE       0     0     0
	    disk2         OFFLINE      0     0     0

errors: No known data errors

pool: data2
state: ONLINE
scan: none requested
config:

	NAME            STATE     READ WRITE CKSUM
	data2           ONLINE       0     0     0
	  disk2         ONLINE       0     0     0

errors: No known data errors
```

You should therefore see your datasets as follows :

```
$ zfs list
```
```
NAME        USED  AVAIL  REFER  MOUNTPOINT
data       1.28T  1.40T    96K  none
data/data  1.28T  1.40T  1.28T  /data
data2       300K  2.68T    96K  none
```

7) Now, you need to create a new dataset for the new pool (options documented in the man pages):

```
$ zfs create -o compression=lz4 data2/data
$ zfs set atime=off data2/data
$ zfs set mountpoint=/data2 data2/data
```

Which should give you the following results (where we can see both the old and the new datasets):

```
zfs list
data        1.28T  1.40T    96K  none
data/data   1.28T  1.40T  1.28T  /data
data2        420K  2.68T    96K  none
data2/data    96K  2.68T    96K  /data2
```

8) Now it's time to move the data from one of the drive to the other :

```
$ mv /data/* /data2/
```

If you have a lot of data to move you can Ctrl+Z, `bg 1`, `disown %1` in order to detach the process from the current terminal.

9) When the data has been successfully moved over, you can destroy the original pool for good :

```
$ zpool destroy data
```

10) And then re-attach the first disk to the existing new pool :

```
$ zpool attach data2 /dev/mapper/disk2 /dev/mapper/disk1
```

Your new pool in mirror mode should be looking good, and the process of **resilvering** should have started :

```
$ zpool status
```
```
pool: data2
state: ONLINE
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
scan: resilver in progress since Sun Apr 26 12:55:40 2015
    220M scanned out of 1.28T at 36.7M/s, 10h9m to go
    216M resilvered, 0.02% done
config:

	NAME              STATE     READ WRITE CKSUM
	data2             ONLINE       0     0     0
	  mirror-0        ONLINE       0     0     0
	    disk2         ONLINE       0     0     0
	    disk1         ONLINE       0     0     0  (resilvering)

errors: No known data errors
```

That's it, hoping this works out for you.


---
References :
https://www.mail-archive.com/zfs-discuss@opensolaris.org/msg11334.html
https://pthree.org/2012/12/04/zfs-administration-part-i-vdevs/



