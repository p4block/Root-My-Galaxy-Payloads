# a36xq-A366BXXU1AYB9

Target profile for the EU Galaxy A36 5G (SM-A366B, Qualcomm SM6475 "parrot"):

```text
model: SM-A366B
device: a36xq
firmware: A366BXXU1AYB9
display build: AP3A.240905.015.A2.A366BXXU1AYB9
fingerprint: samsung/a36xqnaeea/a36xq:15/AP3A.240905.015.A2/A366BXXU1AYB9_OXM1AYB9:user/release-keys
kernel: 6.6.46-android15-8-30526735-abogkiA366BXXU1AYB9-4k
page size: 4096
```

All offsets and the P0 fingerprint table are derived from the exact AYB9
kernel `Image` (SHA-256 `60fcc53e961eb40f291a88718c429b95651eccb10eba0868b741ea0af1a62c3d`,
36,694,528 bytes, recovered from `AP_A366BXXU1AYB9_...tar.md5` ->
`boot.img.lz4` -> `boot.img` -> `Image`). The recovered `vmlinux-to-elf` base
is `0xffffffc080000000`.

Key derived values:

- `SLIDE_TRACEFS_EVENT_ID = 109`: read authoritatively from the device at
  `/sys/kernel/tracing/events/sched/sched_blocked_reason/id`; static
  derivation agrees (`__TRACE_LAST_TYPE` 20 + zero-based index 89).
- `SLIDE_TRACEFS_WORKER_CALLER_OFF = 0x000d8bbc`: instruction after the
  blocking `bl schedule` in `worker_thread` (`0xffffffc0800d8bb8`).
- `P0_KERNEL_PHYS_LOAD = 0xa8000000`, `P0_PHYS_OFFSET = 0x80000000`:
  Qualcomm ABL (`abl.elf` -> LZMA FV -> `LinuxLoader` PE, FFS GUID
  `f536d559-459f-48fa-8bbc-43b554ecae8d`) carries the same ARM64 load
  constants `0x00080000`/`0x05600000` as the documented S928U1 DZF2 ABL; the
  OEM load offset `0x28000000` matches the device-tested same-SoC
  `a36xq-A366WVLS3AYG1` profile.
- `SKB_DATA_DELTA = -0x1000`: target BTF gives
  `sizeof(struct skb_shared_info) == 320` (0x140), so `SKB_MAX_HEAD(0)` is
  `0xec0` and a 64 KiB `unix_stream_sendmsg` first order-3 fragment lands at
  page offset `0x1000` after page-aligning the fragment length.
- Layout values are byte-identical to the A366W sibling: fop offsets
  (`open` 0x68, `release` 0x78, `unlocked_ioctl` 0x48, `splice_read` 0xb8,
  `show_fdinfo` 0xd8), `task_struct` PI chain (0x90c/0x920/0x930/0x938),
  expanded `rt_mutex_waiter` (size 0x70, `lock` 0x58), `miscdevice.fops`
  0x10.

Generate the 32-row fingerprint table from the raw kernel Image with:

```sh
tools/generate_p0_fingerprint.pl Image 0x1f0000 p0_fingerprint.h
```

Build with Android NDK r29:

```sh
make TARGET=a36xq-A366BXXU1AYB9 ANDROID_NDK_HOME=/path/to/android-ndk
```

Hardware validation on SM-A366B has not been performed yet; the trace event ID
was captured from the live device and the ABL/physical-load derivation is
static.
