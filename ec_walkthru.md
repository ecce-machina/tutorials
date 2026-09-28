# Lustre Erasure Coding Demo

This walkthrough demonstrates the experimental erasure coding (EC) support currently present in Lustre `master`.

The demo creates a **2+1 EC layout**:

- 2 data stripes
- 1 parity stripe
- 3 OSTs total

> **Warning**
>
> Erasure coding is currently incomplete and is not intended for production use. The Lustre CLI itself requires explicit acknowledgement of this limitation through `LFS_EC_OK=yes`.

## Environment

This demo was tested with:

```text
Lustre version: 2.17.58_62_g550451c
```
using https://github.com/ecce-machina/lustre_lab on GCP, you need to set the packer script to using source so it'll download master and use this instead of built RPMs.

Topology:

```text
MGS/MDS
   |
   +---- OST0000
   +---- OST0001
   +---- OST0002
   |
   +---- Lustre client
```

A `2+1` layout requires at least three OSTs because each raidset needs two data stripes and one parity stripe on distinct OSTs.

## 1. Enable FLR/EC support on the client

EC capability advertisement is disabled by default.

Check the client module parameter:

```bash
cat /sys/module/lustre/parameters/llite_enable_flr_ec
```

If it returns:

```text
0
```

enable it:

```bash
echo 1 > /sys/module/lustre/parameters/llite_enable_flr_ec
```

Verify:

```bash
cat /sys/module/lustre/parameters/llite_enable_flr_ec
```

Expected:

```text
1
```

In the Lustre source, this parameter controls whether the client requests the `OBD_CONNECT2_FLR_EC` capability:

```c
if (llite_enable_flr_ec)
        data->ocd_connect_flags2 |= OBD_CONNECT2_FLR_EC;
```

## 2. Enable FLR/EC support on the MDT

The MDT independently controls whether it accepts the EC capability.

On the MDS:

```bash
cat /sys/module/mdt/parameters/mdt_enable_flr_ec
```

Enable it if necessary:

```bash
echo 1 > /sys/module/mdt/parameters/mdt_enable_flr_ec
```

Verify:

```bash
cat /sys/module/mdt/parameters/mdt_enable_flr_ec
```

Expected:

```text
1
```

The MDT connection handling code only advertises `OBD_CONNECT2_FLR_EC` when this parameter is enabled:

```c
if (mdt_enable_flr_ec)
        supported2 |= OBD_CONNECT2_FLR_EC;

data->ocd_connect_flags2 &= supported2;
```

There is no corresponding `enable_flr_ec` module parameter on the OSS nodes in this build.

## 3. Reconnect the client

The EC capability is negotiated when the client connects to the MDT.

If the filesystem was already mounted when the module parameters were changed, unmount and remount it:

```bash
umount /mnt/lustre

mount -t lustre 10.10.0.10@tcp:/lustrefs /mnt/lustre
```

Verify that EC was negotiated:

```bash
lctl get_param mdc.*.connect_flags | grep -Ei 'flr|ec'
```

Expected output includes:

```text
flr
flr_ec
```

If `flr_ec` is absent, EC was not successfully negotiated with the MDT.

## 4. Enable EC on the mounted client

Negotiating `flr_ec` is not sufficient by itself.

The mounted llite filesystem also has an EC runtime switch. Check it with:

```bash
lctl get_param llite.*.enable_erasure_coding
```

Enable it:

```bash
lctl set_param llite.*.enable_erasure_coding=1
```

This step is important.

Without it, an EC `setstripe` may fail even though `flr_ec` appears in the negotiated MDC connect flags:

```text
lfs setstripe: cannot create composite file 'ecfile3':
Inappropriate ioctl for device (25)
```

The upstream `sanity-ec.sh` test suite performs the same operation:

```bash
lctl set_param llite.*.enable_erasure_coding=1
```

## 5. Acknowledge experimental EC support

The current `lfs` utility deliberately prevents EC operations unless the experimental status has been acknowledged.

Set:

```bash
export LFS_EC_OK=yes
```

Without this variable, `lfs` reports that erasure coding is incomplete and returns `EOPNOTSUPP`.

## 6. Create a 2+1 EC file

Create a directory for the demo:

```bash
mkdir -p /mnt/lustre/ec-demo
cd /mnt/lustre/ec-demo
```

Create a file with two data stripes and one parity stripe:

```bash
lfs setstripe --ec 2+1 ecfile
```

The equivalent explicit stripe-count syntax is:

```bash
lfs setstripe --ec-stripe-count 2+1 ecfile
```

## 7. Inspect the EC layout

Run:

```bash
lfs getstripe -v ecfile
```

A successful `2+1` layout contains two linked components.

The first is the data component:

```text
lcme_mirror_id:      1
lcme_flags:          init

lmm_stripe_count:    2
lmm_pattern:         raid0
```

For example:

```text
OST0001 -> data stripe
OST0000 -> data stripe
```

The second component contains parity:

```text
lcme_mirror_id:      2
lcme_flags:          init,parity

lcme_dstripe_count:  2
lcme_cstripe_count:  1
lcme_ec:             2+1
lcme_ec_raidset_count: 1
```

For example:

```text
OST0002 -> parity stripe
```

Conceptually:

```text
                 ecfile
                    |
          +---------+---------+
          |                   |
     Data component      Parity component
          |                   |
      +---+---+               |
      |       |               |
   OST0001 OST0000         OST0002
     data    data            parity
```

The exact OST assignment can vary; the important property is that a `2+1` raidset contains two data stripes and one parity stripe on distinct OSTs.

## 8. Write test data

Create some data large enough to exercise multiple EC chunks:

```bash
dd if=/dev/urandom of=ecfile bs=1M count=64 status=progress
```

Record a checksum:

```bash
sha256sum ecfile
```

Save the checksum for comparison during recovery testing.

Inspect the layout again after the write:

```bash
lfs getstripe -v ecfile
```

## 9. Test degraded reads

The interesting EC demonstration is reading the file with one member of the raidset unavailable.

First determine which OSTs contain the data and parity objects:

```bash
lfs getstripe -v ecfile
```

Example:
```
sha256sum ecfile
af654c5ea2e8b63b770fcade57325135ff2d126248dfe32c1735c564932ae888  ecfile

lfs getstripe -v ecfile3
ecfile3
composite_header:
  lcm_magic:         0x0BD60BD0
  lcm_size:          264
  lcm_flags:         ro
  lcm_layout_gen:    2
  lcm_mirror_count:  2
  lcm_entry_count:   2
components:
  - lcme_id:             65537
    lcme_mirror_id:      1
    lcme_flags:          init
    lcme_mirror_link_id: 0x2
    lcme_extent.e_start: 0
    lcme_extent.e_end:   EOF
    lcme_offset:         128
    lcme_size:           80
    sub_layout:
      lmm_magic:         0x0BD10BD0
      lmm_seq:           0x200000404
      lmm_object_id:     0x2
      lmm_fid:           [0x200000404:0x2:0x0]
      lmm_stripe_count:  2
      lmm_stripe_size:   4194304
      lmm_pattern:       raid0
      lmm_layout_gen:    0
      lmm_stripe_offset: 1
      lmm_objects:
      -   0: { l_ost_idx:   1, l_fid: [0x280000400:0x2:0x0] }
      -   1: { l_ost_idx:   0, l_fid: [0x2c0000400:0x2:0x0] }

  - lcme_id:             131074
    lcme_mirror_id:      2
    lcme_flags:          init,parity
    lcme_mirror_link_id: 0x1
    lcme_dstripe_count:  2
    lcme_cstripe_count:  1
    lcme_ec:             2+1
    lcme_ec_raidset_count: 1
    lcme_ec_raidsets:
      - 0: { ec_data_count: 2, data_stripes: "0-1", parity_stripes: "0-0" }
    lcme_extent.e_start: 0
    lcme_extent.e_end:   EOF
    lcme_offset:         208
    lcme_size:           56
    sub_layout:
      lmm_magic:         0x0BD10BD0
      lmm_seq:           0x200000404
      lmm_object_id:     0x2
      lmm_fid:           [0x200000404:0x2:0x0]
      lmm_stripe_count:  1
      lmm_stripe_size:   4194304
      lmm_pattern:       raid0,parity
      lmm_layout_gen:    0
      lmm_stripe_offset: 2
      lmm_objects:
      -   0: { l_ost_idx:   2, l_fid: [0x240000400:0x3:0x0] }
```

Indicates that:
```
OST0 = data
OST1 = data
OST2 = parity
```

Then make one OST unavailable using an appropriate test mechanism for the lab.

We'll do OST0, only on the client

```
lctl dl | grep OST0000
  6 UP osc lustrefs-OST0000-osc-ffff8e2e80edf800 766e0ed5-afaa-476f-bd76-ec79e84bdfc2 5
lctl --device lustrefs-OST0000-osc-ffff8e2e80edf800 deactivate
lctl dl | grep OST0000
  6 IN osc lustrefs-OST0000-osc-ffff8e2e80edf800 766e0ed5-afaa-476f-bd76-ec79e84bdfc2 5
```

Read the file again:

```bash
sha256sum ecfile
af654c5ea2e8b63b770fcade57325135ff2d126248dfe32c1735c564932ae888  ecfile
```

If EC recovery succeeds, the checksum should match the checksum recorded before the failure.

This demonstrates the basic EC property:

```text
2 data + 1 parity

       loss of one stripe
              |
              v
 surviving stripes + parity
              |
              v
       reconstructed data
```

Restore the OST after the degraded-read test before proceeding with additional experiments.

## Troubleshooting

### `invalid EC stripe count`

This is a userspace syntax error.

For example:

```bash
lfs setstripe --ec-stripe-count 2 ecfile
```

is invalid.

Specify both data and parity counts:

```bash
lfs setstripe --ec-stripe-count 2+1 ecfile
```

### `Inappropriate ioctl for device (25)`

This is `ENOTTY`.

Check all three EC enablement conditions:

```bash
cat /sys/module/lustre/parameters/llite_enable_flr_ec
```

should return:

```text
1
```

On the MDS:

```bash
cat /sys/module/mdt/parameters/mdt_enable_flr_ec
```

should return:

```text
1
```

The mounted client must also have:

```bash
lctl get_param llite.*.enable_erasure_coding
```

set to:

```text
1
```

Finally, verify negotiation:

```bash
lctl get_param mdc.*.connect_flags | grep -Ei 'flr|ec'
```

which should include:

```text
flr_ec
```

### EC warning from `lfs`

Set:

```bash
export LFS_EC_OK=yes
```

This variable exists because the current implementation is explicitly marked incomplete.

## Source references

Useful places in the Lustre source tree for understanding the current implementation:

```text
lustre/llite/llite_lib.c
lustre/llite/super25.c
lustre/mdt/mdt_handler.c
lustre/mdt/mdt_mds.c
lustre/utils/lfs.c
lustre/tests/sanity-ec.sh
lustre/tests/test-framework.sh
include/uapi/linux/lustre/lustre_idl.h
```

Of particular interest are the capability definitions:

```c
#define OBD_CONNECT2_FLR_EC     0x10000000000ULL /* parity support */
#define OBD_CONNECT2_FLR_EC_WR  0x80000000000ULL /* write EC support */
```

and the experimental userspace guard in `lfs.c`, which currently notes that the EC implementation is incomplete and is expected to be removed for the Lustre 2.18 release.

## Minimal setup summary

For an already configured three-OST test filesystem, the essential setup is:

**MDS:**

```bash
echo 1 > /sys/module/mdt/parameters/mdt_enable_flr_ec
```

**Client, before reconnecting:**

```bash
echo 1 > /sys/module/lustre/parameters/llite_enable_flr_ec
```

Remount the filesystem, then:

```bash
lctl set_param llite.*.enable_erasure_coding=1
export LFS_EC_OK=yes
```

Create and inspect the EC file:

```bash
lfs setstripe --ec 2+1 ecfile
lfs getstripe -v ecfile
```

At this point the filesystem has a genuine `2+1` EC layout consisting of two data stripes and one parity stripe.
