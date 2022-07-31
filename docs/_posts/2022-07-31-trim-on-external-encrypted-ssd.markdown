---
layout:         post
title:          "TRIM on an external encrypted SSD"
date:           2022-07-31 04:29:55 +0300
#modified_date: 2022-07-31 04:29:55 +0300
tags:           ssd trim cryptsetup luks
---

TRIM commands are not (currently) transparently passed through all the layers by
default, so `fstrim` may not work as expected. Here is how it may look like:

```
$ lsblk --discard /dev/sda
NAME                                          DISC-ALN DISC-GRAN DISC-MAX DISC-ZERO
sda                                                  0        0B       0B         0
`-sda1                                               0        0B       0B         0
  `-luks-12345678-90ab-cdef-1234-567890abcdef        0        0B       0B         0
```

A zero in `DISC-GRAN` column means than discard is not supported at that level.

Set `provisioning_mode` to `unmap` to enable TRIM passing for `sda`:
```
# echo unmap > /sys/block/sda/device/scsi_disk/*/provisioning_mode
```

If `provisioning_mode` set successfully, the output of `lsblk --discard` will change:

```
$ lsblk --discard /dev/sda
NAME                                          DISC-ALN DISC-GRAN DISC-MAX DISC-ZERO
sda                                                  0        4K       4G         0
`-sda1                                               0        4K       4G         0
  `-luks-12345678-90ab-cdef-1234-567890abcdef        0        0B       0B         0
```

Now LUKS needs to be asked to pass TRIM too:

```
# cryptsetup refresh luks-12345678-90ab-cdef-1234-567890abcdef --allow-discards
```

Cryptsetup will ask the passphrase for the volume. If changes were successful,
the output of `lsblk --discard` will look like this:

```
$ lsblk --discard /dev/sda
NAME                                          DISC-ALN DISC-GRAN DISC-MAX DISC-ZERO
sda                                                  0        4K       4G         0
`-sda1                                               0        4K       4G         0
  `-luks-12345678-90ab-cdef-1234-567890abcdef        0        4K       4G         0
```

The definition of success is to have non-zeroes in all rows in `DISC-GRAN` column.

---

### What could go wrong

`provisioning_mode` parameter may not exist for the particular enclosure. It
seems that the enclosure must use USB Attached SCSI protocol for TRIM passing to
work. It therefore a good sign if `uas` is mentioned in `dmesg` output near
information about new connected device. Some controllers of external enclosures
are supposedly having proprietary extensions that help to issue TRIM command to
the actual storage device. I haven't seen any successful attempts to use those
yet.

### Automation

TODO
