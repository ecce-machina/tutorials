# Following a Lustre File into ldiskfs

While trying to get a better handle on ldiskfs for work, I decided to follow a file from the Lustre client down to its MDT inode and OST object. I created an empty file, inspected both targets, wrote 8 MiB to it, and then looked at what changed.

The goal is to build a concrete mental model of the relationship between:

- the file's client-visible FID;
- its namespace inode and layout metadata on the MDT;
- its separate object FID and data inode on the OST; and
- the Size-on-MDT (`trusted.som`) summary.

The examples come from a small `lustrefs` lab filesystem. Device names, target
indices, FIDs, object IDs, and backend paths will differ on another filesystem.

> **Lab safety:** The commands below use `debugfs` read-only. Do not use
> `debugfs -w` against Lustre backing devices. Direct inspection of live backing
> devices should be confined to a disposable lab or performed under an approved
> support procedure. Even after `sync`, the output is a point-in-time view, not
> a transactional snapshot.

## Mental model

A regular Lustre file is not one ordinary ldiskfs inode containing both its
namespace metadata and file data.

```text
Client pathname
    |
    v
MDT inode: name, ownership, timestamps, layout, logical file FID
    |
    | trusted.lov identifies one or more OST objects
    v
OST inode(s): allocated blocks and the actual file data
```

In this example there is one stripe, so the logical file has one OST object:

| Role | FID |
|---|---|
| Logical file on the MDT | `[0x200000404:0x5:0x0]` |
| Data object on OST0001 | `[0x280000400:0x4:0x0]` |

These are related, but they are deliberately different identities.

## 1. Identify the client mount and targets

On the client:

```console
# findmnt -t lustre -o TARGET,SOURCE
TARGET       SOURCE
/mnt/lustre  10.10.0.10@tcp:/lustrefs

# lfs df
UUID                    1K-blocks      Used  Available Use% Mounted on
lustrefs-MDT0000_UUID     58189732      1716   52928756   1% /mnt/lustre[MDT:0]
lustrefs-OST0000_UUID: Cannot send after transport endpoint shutdown
lustrefs-OST0001_UUID    102165532     66932   96839336   1% /mnt/lustre[OST:1]
lustrefs-OST0002_UUID    102165532      1396   96904872   1% /mnt/lustre[OST:2]
```

OST0000 happened to be unavailable in this lab. The file below is explicitly
placed on OST0001.

## 2. Create a directory and an empty file on OST0001

Set a one-stripe default layout on the directory, starting at target index 1:

```bash
mkdir -p /mnt/lustre/ldiskfs
lfs setstripe -c 1 -i 1 /mnt/lustre/ldiskfs
touch /mnt/lustre/ldiskfs/hello
```

The directory's `trusted.lov` is a **default layout template**. New files in the
directory inherit that policy unless another layout is requested explicitly.
The regular file's `trusted.lov`, by contrast, describes its actual layout.

Inspect the new file from the client:

```console
# lfs path2fid /mnt/lustre/ldiskfs/hello
[0x200000404:0x5:0x0]

# lfs getstripe -v /mnt/lustre/ldiskfs/hello
/mnt/lustre/ldiskfs/hello
lmm_magic:         0x0BD10BD0
lmm_seq:           0x200000404
lmm_object_id:     0x5
lmm_fid:           [0x200000404:0x5:0x0]
lmm_stripe_count:  1
lmm_stripe_size:   4194304
lmm_pattern:       raid0
lmm_layout_gen:    0
lmm_stripe_offset: 1
        obdidx          objid          objid          group
             1              4             0x4      0x280000400
```

The important values are:

| Field | Value | Meaning |
|---|---:|---|
| File FID | `[0x200000404:0x5:0x0]` | Logical identity stored on the MDT |
| Stripe count | `1` | One OST object |
| Stripe size | `4194304` | 4 MiB |
| `obdidx` | `1` | OST0001 |
| OST sequence/group | `0x280000400` | First component of the OST object's FID |
| OST object ID | `4` | Second component of the OST object's FID |

Thus, the data object's FID is `[0x280000400:0x4:0x0]`.

An initial `stat` reports a zero-length file:

```console
# stat /mnt/lustre/ldiskfs/hello
  File: /mnt/lustre/ldiskfs/hello
  Size: 0             Blocks: 0          IO Block: 4194304 regular empty file
Device: 481510a2h/1209340066d  Inode: 144115205322833925  Links: 1
```

The inode number shown by the Lustre client is not the inode number of either
backend ldiskfs inode.

## 3. Inspect the file on the MDT

On the MDS, the MDT is mounted from `/dev/sdb`:

```console
# findmnt -t lustre -o TARGET,SOURCE
TARGET     SOURCE
/mnt/mdt0  /dev/sdb

# lsblk -f /dev/sdb
NAME FSTYPE FSVER LABEL               UUID
sdb  ext4   1.0   lustrefs:MDT0000    193aa567-482d-45ef-ae47-0b001c6d9b40
```

Lustre presents the namespace below the special `/ROOT` directory on an
ldiskfs MDT. Inspect the file without modifying the device:

```bash
sync
debugfs -R 'stat /ROOT/ldiskfs/hello' /dev/sdb
debugfs -R 'ea_list /ROOT/ldiskfs/hello' /dev/sdb
```

Before data is written, the relevant output is:

```text
Inode: 35127305   Type: regular    Mode: 0644
Size: 0
Links: 1   Blockcount: 0

Extended attributes:
  lma: fid=[0x200000404:0x5:0x0] compat=0 incompat=0
  trusted.lov (56) = ...
  linkea: idx=0 parent=[0x200000404:0x4:0x0] name='hello'
  trusted.som (24) = 04 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
                     00 00 00 00 00 00 00 00
EXTENTS:
```

There are no data extents on the MDT inode.

### What the MDT extended attributes mean

| Attribute | Purpose |
|---|---|
| `lma` | Lustre Metadata Attribute: records this inode's own Lustre identity, including its FID. |
| `trusted.lov` | For this regular file, records its layout and the OST object mapping. On a directory, it can instead hold the default layout inherited by new files. |
| `linkea` | Records namespace link information. Here, the parent directory FID is `[0x200000404:0x4:0x0]` and the entry name is `hello`. |
| `trusted.som` | Size-on-MDT: a cached summary of logical size and allocated blocks, subject to its validity state. It is not the file's data. |

Not every Lustre inode has both `trusted.lov` and `trusted.som`. The applicable
attributes depend on the object type and state. The `lma` identifies the object
itself; a regular file with a materialized layout has `trusted.lov`; a directory
may have `trusted.lov` as a default layout; and eligible regular files may carry
SOM information.

Notice the two different meanings of “parent” encountered later:

- the MDT `linkea` parent is the containing namespace directory;
- the OST `fid` attribute's parent is the logical MDT file that owns the stripe.

## 4. Find the OSS that currently serves OST0001

`lctl dl` identifies the connected target, but it does not directly give the
server hostname. Query the OSC import on the client:

```bash
lctl get_param -n 'osc.lustrefs-OST0001-osc-*.import' \
    | grep -E 'current_connection|failover_nids'
```

This reveals the active server NID. In a Google Cloud lab, map its IP address to
an instance name with:

```bash
gcloud compute instances list \
    --filter='networkInterfaces.networkIP=10.10.0.12'
```

In this run, OST0001 was served by `lustre-oss2`.

On that OSS, confirm the backing device rather than assuming it:

```console
# findmnt -t lustre -o TARGET,SOURCE
TARGET     SOURCE
/mnt/ost1  /dev/sdb

# lsblk -f /dev/sdb
NAME FSTYPE FSVER LABEL               UUID
sdb  ext4   1.0   lustrefs:OST0001    a27b98e9-6f03-48a6-aa43-71f9329c6f46
```

## 5. Special section: map an OST object ID to its ldiskfs path

For the legacy ldiskfs OST object directory layout, the path is:

```text
/O/<sequence>/d<bucket>/<object-id>
```

The bucket is selected with:

```text
bucket = object-id & (subdirectory-count - 1)
```

The usual subdirectory count is 32, giving `object-id & 31`. Lustre stores the
actual count in the target metadata, so do not treat 32 as an eternal on-disk
format guarantee.

For this stripe:

```text
sequence/group = 0x280000400
object-id      = 4
bucket         = 4 & 31 = 4
```

The sequence directory is written without the `0x` prefix, so the path is:

```text
/O/280000400/d4/4
```

In general:

```text
/O/<sequence>/d<object-id & (subdirectory-count - 1)>/<object-id>
```

Older output and documentation may call the sequence a **group**.

## 6. Inspect the empty OST object

On the OSS hosting OST0001:

```bash
sync
debugfs -R 'stat /O/280000400/d4/4' /dev/sdb
debugfs -R 'ea_list /O/280000400/d4/4' /dev/sdb
```

Before the write:

```text
Inode: 105   Type: regular    Mode: 07666
Generation: 3622796191
Size: 0
Links: 1   Blockcount: 0

Extended attributes:
  lma: fid=[0x280000400:0x4:0x0] compat=8 incompat=0
  fid: parent=[0x200000404:0x5:0x0] stripe=0 stripe_size=4194304
       stripe_count=1 layout_version=0 range=0
EXTENTS:
```

This establishes both directions of the relationship:

- the MDT file's LOV layout points forward to the OST object;
- the OST object's `fid` attribute points back to the owning MDT file FID and
  says that this is stripe 0.

The OST inode's creation time predated the file's creation time in this run.
That is consistent with Lustre's use of precreated OST objects: the backend
object may exist before it is assigned to a logical file.

## 7. Write 8 MiB through the Lustre client

Back on the client:

```bash
dd if=/dev/zero of=/mnt/lustre/ldiskfs/hello \
   bs=1M count=8 conv=fsync status=progress

stat /mnt/lustre/ldiskfs/hello
```

The result was:

```text
8+0 records in
8+0 records out
8388608 bytes (8.4 MB, 8.0 MiB) copied

Size: 8388608       Blocks: 16384      IO Block: 4194304 regular file
Inode: 144115205322833925
```

`stat` reports blocks in 512-byte units, so `16384 * 512 = 8388608` bytes.

## 8. Inspect the OST object after the write

Run the same read-only inspection on the OSS:

```bash
sync
debugfs -R 'stat /O/280000400/d4/4' /dev/sdb
debugfs -R 'ea_list /O/280000400/d4/4' /dev/sdb
```

Now the object contains the data allocation:

```text
Inode: 105   Type: regular    Mode: 0666
Generation: 3622796191
Size: 8388608
Links: 1   Blockcount: 16384

Extended attributes:
  lma: fid=[0x280000400:0x4:0x0] compat=8 incompat=0
  fid: parent=[0x200000404:0x5:0x0] stripe=0 stripe_size=4194304
       stripe_count=1 layout_version=0 range=0
EXTENTS:
(0-2047):482304-484351
```

The logical range contains 2048 filesystem blocks. With 4 KiB ldiskfs blocks:

```text
2048 * 4096 = 8388608 bytes = 8 MiB
```

It maps to one contiguous physical extent in this particular run.

The backend mode changed from `07666` to `0666` when the object transitioned
through its initial lifecycle. Those backend bits are internal implementation
state; they should not be interpreted as the client-visible POSIX mode.

## 9. Revisit the MDT and decode `trusted.som`

On the MDS:

```bash
sync
debugfs -R 'stat /ROOT/ldiskfs/hello' /dev/sdb
debugfs -R 'ea_list /ROOT/ldiskfs/hello' /dev/sdb
```

After the write, the MDT's backend inode still has no file-data allocation:

```text
Inode: 35127305   Type: regular    Mode: 0644
Size: 0
Links: 1   Blockcount: 0
EXTENTS:
```

Its `trusted.som` value is now:

```text
04 00 00 00 00 00 00 00
00 00 80 00 00 00 00 00
00 40 00 00 00 00 00 00
```

This is the raw 24-byte extended-attribute value rendered as hexadecimal—not
an ldiskfs data block. Depending on available xattr space and filesystem
features, the bytes may be stored in the inode or in external xattr storage.

The value corresponds to three little-endian 64-bit fields:

```c
struct lustre_som_attrs {
        __u64 lsa_valid;
        __u64 lsa_size;
        __u64 lsa_blocks;
};
```

Decoded:

| Offset | Bytes | Field | Value |
|---:|---|---|---:|
| 0 | `04 00 00 00 00 00 00 00` | `lsa_valid` | `0x4` |
| 8 | `00 00 80 00 00 00 00 00` | `lsa_size` | `0x800000` = 8,388,608 bytes |
| 16 | `00 40 00 00 00 00 00 00` | `lsa_blocks` | `0x4000` = 16,384 blocks |

The SOM size and block count agree with the client and OST observations, but
they remain cached summary metadata. The actual data blocks are on the OST.

## 10. Before-and-after comparison

| Observation point | Before write | After 8 MiB write |
|---|---:|---:|
| Client logical size | 0 | 8,388,608 |
| Client `st_blocks` | 0 | 16,384 |
| MDT backend inode size | 0 | 0 |
| MDT backend data extents | none | none |
| MDT SOM cached size | 0 | 8,388,608 |
| MDT SOM cached blocks | 0 | 16,384 |
| OST backend inode | 105 | 105 |
| OST object size | 0 | 8,388,608 |
| OST object blocks | 0 | 16,384 |
| OST data extents | none | `(0-2047):482304-484351` |

## Takeaways

1. The client-visible FID identifies the logical file represented on the MDT.
2. Each stripe is a separate object with its own OST-side FID.
3. The MDT file's `trusted.lov` maps the logical file to its OST object or
   objects.
4. The OST object's `fid` attribute provides a backreference to the owning MDT
   file and identifies the stripe number.
5. The MDT backend inode does not grow to the file's logical size and does not
   acquire the file's data extents.
6. The OST inode holds the real allocation and data extents.
7. `trusted.som` lets the MDT cache size and block information without storing
   the file data locally.
8. A multi-stripe file keeps one MDT FID but has multiple distinct OST object
   FIDs—one per component object.

## References

- [Lustre `osd_compat.c`: legacy OST object directory layout](https://lustre.software/repos/master/?path=lustre/osd-ldiskfs/osd_compat.c)
- [Lustre Layout Enhancement High-Level Design](https://wiki.lustre.org/Layout_Enhancement_High_Level_Design)
- [Introduction to Lustre Object Storage Devices](https://wiki.lustre.org/Introduction_to_Lustre_Object_Storage_Devices_%28OSDs%29)
- [`debugfs(8)` manual page](https://man7.org/linux/man-pages/man8/debugfs.8.html)

