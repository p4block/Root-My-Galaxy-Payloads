# SM-A366B / A366BXXU1AYB9 porting record

This file records the port of the EU Galaxy A36 5G (Qualcomm SM6475
"parrot", device `a36xq`) to the CVE-2026-43499 shared exploit. Values from
other devices are not reused; the sibling Canadian profile
`a36xq-A366WVLS3AYG1` is same-SoC and same-kernel-branch but a different
build, so every symbol was re-derived.

## Firmware identity

| Field | Value |
| --- | --- |
| Model | `SM-A366B` (EUX sales code) |
| AP/PDA | `A366BXXU1AYB9` |
| Device codename | `a36xq` |
| Android build ID | `AP3A.240905.015.A2` |
| Build fingerprint | `samsung/a36xqnaeea/a36xq:15/AP3A.240905.015.A2/A366BXXU1AYB9_OXM1AYB9:user/release-keys` |
| Kernel release | `6.6.46-android15-8-30526735-abogkiA366BXXU1AYB9-4k` |
| Page size | 4096 |

## Kernel chain

| Object | Size (bytes) | SHA-256 |
| --- | ---: | --- |
| Raw kernel `Image` | 36,694,528 | `60fcc53e961eb40f291a88718c429b95651eccb10eba0868b741ea0af1a62c3d` |

The AP archive member `boot.img.lz4` decompresses to an Android boot image
(header version 4, 4096-byte page) whose 36,694,528-byte payload is already
an uncompressed ARM64 Image (`text_offset=0`, `image_size=0x2550000`,
`flags=0xa`). `vmlinux-to-elf` recovered a symbolized ELF with base
`0xffffffc080000000`. A single validated raw BTF blob spans
`[0x171edfc, 0x1cd555c)` (5,990,240 bytes).

## Target symbols (image offsets from `0xffffffc080000000`)

| Use | Symbol | Offset |
| --- | --- | ---: |
| usermode-helper worker | `call_usermodehelper_exec_work` | `0x000d0274` |
| no-op seek | `noop_llseek` | `0x003a4be8` |
| splice read | `copy_splice_read` | `0x003f1798` |
| configfs read | `configfs_read_iter` | `0x0046ccf8` |
| configfs binary write | `configfs_bin_write_iter` | `0x0046d224` |
| ashmem ioctl | `ashmem_ioctl` | `0x00c9cce0` |
| ashmem compat ioctl | `compat_ashmem_ioctl` | `0x00c9d39c` |
| ashmem mmap | `ashmem_mmap` | `0x00c9d3f0` |
| ashmem open | `ashmem_open` | `0x00c9d610` |
| ashmem release | `ashmem_release` | `0x00c9d698` |
| ashmem fdinfo | `ashmem_show_fdinfo` | `0x00c9d724` |
| anonymous pipe ops | `anon_pipe_buf_ops` | `0x01167208` |
| ashmem fops | `ashmem_fops` | `0x01313338` |
| `"nfnetlink_log"` string | `nfulnl_logger.name` target | `0x01628338` |
| allocator caches | `kmalloc_caches` | `0x01694f50` |
| unbound workqueue | `system_unbound_wq` | `0x02109e60` |
| netfilter logger object | `nfulnl_logger` | `0x02111260` |
| logger array | `loggers` | `0x0211111a8` (abs) / `0x0211111a8` |
| initial task | `init_task` | `0x0211cbc0` |
| ashmem misc fops | `ashmem_misc + 0x10` | `0x02278ab0` |
| root task group | `root_task_group` | `0x0230d500` |
| SELinux enforcing | `selinux_state.enforcing` | `0x0234ebb0` |
| boot-ID table data ptr | `random_table[].data` of `boot_id` | `0x022361b0` |
| boot-ID storage | `sysctl_bootid` | `0x023f0768` |

`random_table`'s `boot_id` entry data slot at `0x022361b0` is the only qword
in the Image equal to the target address of `sysctl_bootid`
(`0xffffffc0823f0768`).

## Trace event ID

Read authoritatively from the live device:

```sh
a36xq:/ $ cat /sys/kernel/tracing/events/sched/sched_blocked_reason/id
109
```

Static derivation agrees: the ftrace-events section spans
`__start_ftrace_events` (`0x020cbcb0`) to `__event_sched_blocked_reason`
(`0x020cbf78`) = `0x2c8` bytes = zero-based entry 89, and the Android 6.6
dynamic base is 20, giving `20 + 89 = 109`.

`SLIDE_TRACEFS_WORKER_CALLER_OFF` is `0x000d8bbc`: `worker_thread` executes
`bl schedule` at `0xffffffc0800d8bb8` and the saved return address is the
next instruction at `0xffffffc0800d8bbc`.

## Layout values (target BTF)

```text
file_operations: size 0x108; unlocked_ioctl 0x48; compat_ioctl 0x50;
  mmap 0x58; open 0x68; release 0x78; splice_read 0xb8; show_fdinfo 0xd8
task_struct: usage 0x40; prio 0x84; normal_prio 0x8c;
  sched_task_group 0x348; pi_lock 0x90c; pi_waiters 0x920;
  pi_top_task 0x930; pi_blocked_on 0x938
rt_mutex_waiter: size 0x70; tree 0x00; pi_tree 0x28; task 0x50; lock 0x58;
  wake_state 0x60; ww_ctx 0x68
configfs_buffer: page 0x10; needs_read_fill 0x50; bin_buffer 0x58;
  bin_buffer_size 0x60; cb_max_size 0x64
pool_workqueue: pool 0x00; wq 0x08; work_color 0x10; refcnt 0x18;
  nr_active 0x5c; max_active 0x60
worker_pool: worklist 0x28; nr_idle 0x3c
workqueue_struct.dfl_pwq 0xb0
work_struct: data 0x00; entry 0x08; func 0x18
page: size 0x40; compound_head 0x08; page_type 0x30
skb_shared_info: size 320 (0x140)
```

The `rt_mutex_waiter` layout is the expanded 6.6 form identical to the A366W
sibling. The skb fragment geometry follows the device-tested same-SoC A366W
profile: the first order-3 fragment of a 64 KiB `unix_stream_sendmsg` send
begins at source offset `0xe80` on this platform (the `-0x1000` derivation
from the 6.1 S928U1 procedure does not apply to this 6.6 kernel and caused a
kernel panic during hardware validation):

```c
#define SKB_DATA_DELTA (-0xe80LL)
```

## Physical load addresses

The BL archive's `abl.elf` (decompressed SHA-256
`bd46c019c15c65810814b4c3345e3b16f0351bd0f4e626a0de6c8df675fafbd8`) is an
ELF32 ARM container holding an LZMA-compressed UEFI firmware volume
(GUID `ee4e5898-3914-4259-9d6e-dc7bd79403cf`). The `LinuxLoader` PE32+
application (FFS GUID `f536d559-459f-48fa-8bbc-43b554ecae8d`) carries the
same ARM64 load constants as the documented S928U1 DZF2 ABL: an
`0x00080000` ARM64 offset paired with a `0x05600000` kernel region size
(literal pool at RVA `0xe30f8`), and it stores the computed entry at the
`Kernel Load Address:` diagnostic's backing slot. The kernel Image
`text_offset` is `0`.

The OEM load offset used by this parrot platform is `0x28000000`, matching
the device-tested same-SoC `a36xq-A366WVLS3AYG1` profile. Therefore:

```c
#define P0_PHYS_OFFSET 0x80000000ULL
#define P0_KERNEL_PHYS_LOAD 0xa8000000ULL
```

## Race/gate parameters

Carried over unchanged from the device-tested same-SoC A366W profile
(`ROUTE_WAIT_SECONDS`, ksnitch tuning, bank layout, oracle slots); these are
SoC/kernel characteristics rather than firmware-specific values.

## Build results

Payload build (`API=33`, NDK r25): the plain `APP_PRELOAD` `-O2` build, same
recipe as the device-proven A366W artifact (the `-Oz` release truncation
changes code timing; on hardware it completed the PI choreography but never
opened the write window):

```text
artifacts/a36xq-A366BXXU1AYB9/cve-2026-43499-app.so
size: 129992
SHA-256: b7caec4f2f52447272b4b5681101075714f70e157ec35f4e6a29e3ece65f5d28
```

KernelSU v3.2.5 + `KernelSU-v3.2.5-samsung-kdp-rkp-defex.patch`, built with
DDK `android15-6.6`, release string patched to the exact target value:

```text
vermagic: 6.6.46-android15-8-30526735-abogkiA366BXXU1AYB9-4k SMP preempt mod_unload modversions aarch64
```

Manual-relocation audit (`kernelsu/tools/audit_module_against_target.py`)
against the recovered `vmlinux.elf`: 214 undefined symbols, 0 missing from
the target symbol table, 0 `__versions` entries, 0 CRC mismatches.

Artifacts:

```text
kernelsu/android15-6.6_kernelsu-A366BXXU1AYB9-kdp.ko  (stripped module, for audit)
kernelsu/ksud-A366BXXU1AYB9-kdp                        (late-load binary embedding the module)
```

Hardware execution on SM-A366B remains a separate validation step; the trace
event ID was captured from the live device, but the exploit chain itself has
not yet been executed against this firmware.
