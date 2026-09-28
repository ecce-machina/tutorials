### Mapping a Lustre OST object to its ldiskfs path

For an ldiskfs-backed OST, objects are stored using this structure:

```text
/O/<sequence>/d<bucket>/<object_id>
```

The bucket is calculated as:

```text
bucket = object_id & (subdirectory_count - 1)
```

The usual subdirectory count is 32, producing directories `d0` through `d31`. In that case:

```text
bucket = object_id & 31
```

For example, `lfs getstripe -v` reported:

```text
OST index:  1
object ID:  4
group:      0x280000400
```

Therefore:

```text
bucket = 4 & 31 = 4
path   = /O/280000400/d4/4
```

The `group` field is the object sequence in older Lustre terminology. The sequence directory is written without the `0x` prefix.

The subdirectory count is stored in the OST’s `last_rcvd` data and is normally 32, but it should be confirmed rather than assumed when examining an unfamiliar filesystem.
