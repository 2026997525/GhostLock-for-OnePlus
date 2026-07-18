# GhostLock — OnePlus

[![中文](https://img.shields.io/badge/语言-中文-red)](README_CN.md)

Kernel exploit targeting locked-bootloader OnePlus devices. Uses **CVE-2026-43499** to obtain root access **without unlocking the bootloader or modifying boot.img**.

> **Authorized security research and educational purposes only.**

---

## Table of Contents

- [Vulnerability Overview](#vulnerability-overview)
- [Supported Devices](#supported-devices)
- [Prerequisites](#prerequisites)
- [Building](#building)
- [Usage](#usage)
- [Run Modes](#run-modes)
- [Technical Details](#technical-details)
- [File Structure](#file-structure)
- [Adding New Devices](#adding-new-devices)
- [FAQ](#faq)

---

## Vulnerability Overview

| Item | Detail |
|------|--------|
| **CVE** | CVE-2026-43499 |
| **Type** | Futex PI (Priority Inheritance) Use-After-Free |
| **Scope** | Linux kernel 2.6.39 ~ 7.1 |
| **Fixed in** | Mainline 7.1 (commit `3bfdc63936dd`) |
| **Android Status** | GKI 6.12.x **still vulnerable** |

### Root Cause

The `pselect6` syscall copies `fd_set` to the kernel stack. When combined with the futex PI waiter mechanism, a freed stack frame can be reallocated as a `rt_mutex_waiter` struct. During PI chain traversal, rb-tree rebalancing writes controlled data to arbitrary kernel addresses.

### Chain

```
futex PI UAF (CVE-2026-43499)
  ├─ Forge rt_mutex_waiter object
  ├─ Control kernel stack via pselect/select fd_set layout
  ├─ Trigger rt_mutex PI operation for arbitrary write
  ├─ Write 1: selinux_state.enforcing = 0
  └─ Write 2: cred → init_cred (uid=0, full capabilities)
```

---

## Supported Devices

| Device | Codename | SoC | Kernel | Firmware | Status |
|--------|----------|-----|--------|----------|--------|
| OnePlus Ace 6T | PLR110 | SM8845 (Snapdragon 8s Elite) | `6.12.38-android16-5-...-ab14275539-4k` | ColorOS 16.0.2.403 | ✅ Verified |
| OnePlus Ace 6T | PLR110 | SM8845 (Snapdragon 8s Elite) | `6.12.38-android16-5-...-ab14552068-4k` | ColorOS 16.0.8.301 | ✅ Verified |
| OnePlus 15 | PLK110 | SM8845 (Snapdragon 8s Elite) | `6.12.23-android16-5-...-ab14541642-4k` | — | ✅ Verified |

> Other OnePlus devices on the same SoC family, Android 16, or kernel 6.12.x can be adapted via boot.img offset extraction.

---

## Prerequisites

### ksud (Required for KernelSU)

GhostLock handles privilege escalation. KernelSU installation requires **ksud** (bundled with KMI-specific `kernelsu.ko`):

| Source | Notes |
|--------|-------|
| **ReSukiSU APK** (recommended) | Install [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU); the APK bundles `libksud.so` |
| **CI Release** | Download from [ReSukiSU CI](https://github.com/cctv18/ReSukiSU_CI/releases) (`ksud-aarch64-linux-android.zip`) |

> Without ksud, the exploit still gets uid=0 root shell, but KernelSU won't be installed and `su` won't persist.

---

## Building

### Prerequisites

- Android NDK (r25+)
- Set `ANDROID_NDK_HOME` or `ANDROID_NDK_ROOT`

### Build

```bash
# Default (API 35)
make

# Specify API level
make API=34

# Specify NDK path
NDK=/path/to/android-ndk make
```

### Artifact

`ghostlock` — statically linked ARM64 ELF executable.

---

## Usage

### One-time Setup

```bash
# 1. Enable ADB TCP mode
adb tcpip 5555

# 2. Push ADB key (required for bootstrap mode)
adb push ~/.android/adbkey /data/local/tmp/a/adbkey

# 3. Push the exploit
adb push ghostlock /data/local/tmp/a/e
adb shell chmod 755 /data/local/tmp/a/e
```

> After first success, `resetprop` automatically sets `persist.adb.tcp.port=5555`, allowing fully automatic runs on subsequent reboots.

---

## Run Modes

### Full Exploit (ADB shell context)

```bash
/data/local/tmp/a/e
```

- perf available, precise child task_struct leaking
- Two-stage: W1 disable SELinux → W2 privilege escalation → KernelSU load

### Bootstrap Mode (App context, seccomp restricted)

```bash
/data/local/tmp/a/e --bootstrap
```

1. **Write 1** → disable SELinux
2. Use freed permissions to `setprop` enable ADB TCP 5555
3. Built-in Mini ADB client connects to `127.0.0.1:5555`
4. RSA authentication with pre-pushed key
5. Full exploit execution via ADB shell (no seccomp)

### Write 1 Only

```bash
/data/local/tmp/a/e --write1
```

- Up to 20 attempts
- Useful for debugging or temporary SELinux disable

---

## Technical Details

### 1. Runtime Kernel Matching

Offsets stored in `src/devices/offsets.h` lookup table, keyed by `uname -r`. Program auto-matches at startup; unknown kernels are rejected.

```c
static const struct kernel_offsets known_offsets[] = {
  OFFSETS_ENTRY("6.12.38-android16-5-...-ab14275539-4k", ...),
  OFFSETS_ENTRY("6.12.38-android16-5-...-ab14552068-4k", ...),
  OFFSETS_ENTRY("6.12.23-android16-5-...-ab14541642-4k", ...),
  { .uname_r = NULL }  /* sentinel */
};
```

#### Offset Sources

| Type | Count | Extraction |
|------|-------|------------|
| kallsyms global symbols | 28 | `tools/extract_target.py` |
| BTF struct fields | 57 | `tools/extract_btf.py` |
| Derived values | 9 | Auto-calculated |
| Fixed constants | 12 | Hardcoded |

#### BTF-verified Structs

| Struct | Fields | Purpose |
|--------|--------|---------|
| `task_struct` | 17 | Process descriptor, cred, seccomp |
| `rt_mutex_waiter` | 6 | UAF forge target |
| `cred` | 4 | Credentials, capabilities |
| `seccomp` | 3 | Seccomp filter state |
| `pipe_inode_info` | 11 | Pipe buffer operations |
| `file_operations` | 13 | Fake fops table |
| `mm_struct` | 1 | Memory descriptor owner |

### 2. KASLR Bypass

#### SLIDE Mode — boot_id Leak

When kernel pointers are restricted (`kptr_restrict`), leak the base address via `boot_id` overwritten with a kernel address:

```
Read /proc/sys/kernel/random/boot_id
  └─ UUID contains nfulnl_logger address
      └─ KASLR slide = leaked_addr - image_offset
          └─ kaslr_base
```

#### FOPS/CFI Mode — fops Table Leak

When ashmem device is accessible, use configfs read/write primitives to read function pointers from the ashmem fops table and compute KASLR offset.

### 3. Kernel Heap Spray

Forge kernel objects on order-3 (32KB) pages:

- **Fake `file_operations`** — hijack ashmem miscdevice fops pointer
- **Fake `rt_mutex_waiter`** — simulate PI chain waiter node
- **Fake `task_struct`** — task reference during PI traversal
- **Fake `rt_mutex` (lock)** — correct waiter/owner information

Implemented via **SKB (socket buffer)** + **KernelSnitch**:
- **KernelSnitch** — futex hash collision to leak `mm_struct` addresses
- **SKB spray** — `sendmsg` to fill kernel heap

### 4. Physical Memory R/W (Pipe)

After obtaining KASLR base, use pipe buffers for physical-level memory access:

```
1. Locate pipe buffer in physmap
2. Forge pipe_buffer ops table pointing to known pipe_buf_ops
3. Hijack pipe_buffer.page to target physical address
4. Arbitrary physical read/write via normal pipe operations
```

Supported: `pipe_read64`, `pipe_write64`, `pipe_phys_read_data`, `pipe_phys_write_data`

### 5. Two-Stage Write

#### Write 1 — Disable SELinux

```
Target: selinux_state.enforcing (offset 0x00)
Method: child-node PI write → forge waiter __rb_parent_color
        pointing to selinux_enforcing - 8
        rb-tree rebalance writes 0x00
```

#### Write 2 — Escalate to Root

```
Target: child process cred pointer
Method: 1. fork child → perf_find_task() locate task_struct
        2. calculate cred field offset
        3. child-node PI write → cred = init_cred (uid=0, full caps)
        4. clear seccomp (TIF_SECCOMP + seccomp struct zeroed)
```

Capability readback verification is performed after cred overwrite.

### 6. Built-in Mini ADB Client

`src/core/miniadb.c` — lightweight ADB protocol client for bootstrap mode:

```
1. TCP connect 127.0.0.1:5555
2. A_CNXN → connection request
3. A_AUTH → RSA token challenge
4. dlopen("libcrypto.so") → PEM_read_bio_RSAPrivateKey → RSA_sign
5. A_AUTH (AUTH_SIGNATURE) → signed response
6. A_CNXN → connection established
7. A_OPEN "shell:/data/local/tmp/a/e" → full exploit
```

Supports SHA-1 and SHA-256 signature algorithms.

---

## File Structure

```
ghostlock-oneplus/
├── Makefile                        # Build configuration (NDK cross-compile)
├── README.md                       # Documentation
├── src/
│   ├── core/                       # Core exploit code
│   │   ├── main.c                  # Entry point: two-stage write + root shell
│   │   ├── fops.c                  # FOPS/CFI mode: pselect route, PI write, KASLR leak
│   │   ├── util.c                  # Utilities: heap spray, KASLR, kernel R/W primitives
│   │   ├── slide.c                 # SLIDE mode: boot_id KASLR leak + pselect route
│   │   ├── pipe.c                  # Pipe buffer physical memory R/W (physrw)
│   │   ├── root.c                  # Cred overwrite, seccomp clear, root child mgmt
│   │   ├── miniadb.c              # Built-in ADB client (TCP + RSA auth)
│   │   ├── common.h               # Global macros, structs, declarations, constants
│   │   ├── target.h               # Target memory layout / struct offsets / KASLR params
│   │   ├── offset.h               # Compile-time target config bridge (#include TARGET_CONFIG_H)
│   │   └── kernelsnitch/          # KernelSnitch — mm_struct address leak
│   │       ├── kernelsnitch.h     # Core algorithm: futex hash collision + brute-force
│   │       ├── futex_hash.h       # Futex hash function
│   │       ├── timeutils.h        # CPU timestamp measurement (RDTSC)
│   │       └── utils.h            # Helper macros (pr_info / SYSCHK / ASSERT etc.)
│   └── devices/                   # Device offset tables
│       ├── offsets.h              # Aggregate all device offsets (lookup table + sentinel)
│       ├── ace6t/offsets.h        # OnePlus Ace 6T offsets (2 kernel versions)
│       └── op15/offsets.h         # OnePlus 15 offsets (1 kernel version)
└── tools/                         # Offset extraction toolchain
    ├── extract_target.py          # Extract kallsyms global symbol offsets (28 items)
    └── extract_btf.py             # Extract BTF struct field offsets (57 items)
```

### Core Module Overview

| Module | File(s) | Responsibility |
|--------|---------|----------------|
| **Entry** | `main.c` | CLI parsing, W1/W2 dispatch, bootstrap flow |
| **PI Route** | `fops.c` / `slide.c` | pselect/select stack layout, futex PI chain manipulation |
| **KASLR** | `util.c` / `fops.c` / `slide.c` | Dual-mode bypass: boot_id leak + fops table leak |
| **Heap Spray** | `util.c` | Order-3 page allocation, SKB spray, forge object layout |
| **Phys R/W** | `pipe.c` | Pipe buffer hijacking, arbitrary physical memory access |
| **Priv Esc** | `root.c` | Cred overwrite, capability verification, seccomp clear |
| **ADB** | `miniadb.c` | Lightweight ADB protocol client, RSA auth |
| **Leak Engine** | `kernelsnitch/` | Futex hash collision for mm_struct location |

---

## Adding New Devices

Only the target device's `boot.img` is needed — no root, no device access required.

### Extract Offsets

```bash
# 1. Extract kernel from boot.img
python -c "import struct; d=open('boot.img','rb').read(); \
           open('kernel','wb').write(d[4096:4096+struct.unpack_from('<I',d,8)[0]])"

# 2. Get kallsyms (root: adb shell su -c 'cat /proc/kallsyms' > kallsyms.txt)
#    Or from vmlinux: nm vmlinux > kallsyms.txt

# 3. Extract global symbol offsets
python tools/extract_target.py     # 28 offsets, auto-verified

# 4. Extract struct field offsets
python tools/extract_btf.py kernel  # 57 offsets, auto-verified
```

### Adaptation Steps

1. Create `src/devices/<name>/offsets.h` (see `ace6t/offsets.h`)
2. `#include` it in `src/devices/offsets.h`
3. Update `src/core/target.h` `KIMAGE_TEXT_BASE` and memory layout if needed
4. Rebuild

### Cross-Device Tuning

| Item | Notes |
|------|-------|
| `VA_BITS` | 48 vs 39 → update `target.h` memory layout |
| Timing | Tweak `PSELECT_*` params in `common.h` |
| Ashmem | C vs Rust impl → update extract_target.py symbol matching |
| Secureguard | Non-OnePlus devices may simplify deployment |

---

## FAQ

### Q: No output when running?

Make sure you're running in an **ADB shell**. App context requires `--bootstrap`.

### Q: "no offsets for this kernel"?

Your kernel version is not yet supported. Follow [Adding New Devices](#adding-new-devices) to extract offsets and rebuild.

### Q: Write 1 keeps failing?

- Ensure pinning to the correct CPU core (`CORE` macro)
- Tweak `PSELECT_ENTER_DELAY_USEC` timing
- Check for interfering processes

### Q: Write 2 perf returns 0?

- Check if seccomp blocks `perf_event_open` (use `--bootstrap` in app context)
- Verify KASLR bypass succeeded (`kaslr_base` valid)

### Q: KernelSU not loading?

- Confirm `ksud` exists and is executable
- Check network policy file integrity (`load_policy` fix)

---

## License

**Authorized security research and educational purposes only.**