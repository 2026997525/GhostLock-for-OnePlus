# GhostLock — OnePlus

[![English](https://img.shields.io/badge/Language-English-blue)](README.md)

适用于 OnePlus 锁 Bootloader 设备的内核漏洞利用工具。利用 **CVE-2026-43499** 在**不解锁 Bootloader、不修改 boot.img** 的前提下获取 root 权限。

> **仅限授权的安全研究和教育用途。**

---

## 目录

- [漏洞概述](#漏洞概述)
- [支持设备](#支持设备)
- [前置条件](#前置条件)
- [构建方法](#构建方法)
- [使用方式](#使用方式)
- [运行模式](#运行模式)
- [技术细节](#技术细节)
- [文件结构](#文件结构)
- [适配新设备](#适配新设备)
- [常见问题](#常见问题)

---

## 漏洞概述

| 项目 | 内容 |
|------|------|
| **CVE** | CVE-2026-43499 |
| **类型** | Futex PI（优先级继承）Use-After-Free |
| **影响范围** | Linux 内核 2.6.39 ~ 7.1 |
| **修复版本** | 主线内核 7.1（commit `3bfdc63936dd`） |
| **Android 状态** | GKI 6.12.x **仍未修复** |

### 漏洞原理

`pselect6` 系统调用会将 `fd_set` 拷贝到内核栈上。当与 futex PI waiter 机制结合时，释放后的栈帧可以被重新分配为 `rt_mutex_waiter` 结构体。在 PI 链遍历过程中，红黑树（rb-tree）重平衡操作会向任意内核地址写入可控数据，从而实现**任意内核地址写入原语**。

### 利用链

```
futex PI UAF (CVE-2026-43499)
  ├─ 伪造 rt_mutex_waiter 对象
  ├─ 通过 pselect/select fd_set 布局控制内核栈
  ├─ 触发 rt_mutex PI 操作执行任意写入
  ├─ Write 1: selinux_state.enforcing = 0
  └─ Write 2: cred → init_cred (uid=0, full capabilities)
```

---

## 支持设备

| 设备 | 代号 | SoC | 内核版本 | 固件 | 状态 |
|------|------|-----|---------|------|------|
| 一加 Ace 6T | PLR110 | SM8845 (骁龙 8s Elite) | `6.12.38-android16-5-...-ab14275539-4k` | ColorOS 16.0.2.403 | ✅ 已验证 |
| 一加 Ace 6T | PLR110 | SM8845 (骁龙 8s Elite) | `6.12.38-android16-5-...-ab14552068-4k` | ColorOS 16.0.8.301 | ✅ 已验证 |
| 一加 15 | PLK110 | SM8845 (骁龙 8s Elite) | `6.12.23-android16-5-...-ab14541642-4k` | — | ✅ 已验证 |

> 其他同 SoC 系列、Android 16 / 内核 6.12.x 的一加设备可通过提取 boot.img 适配。

---

## 前置条件

### ksud（KernelSU 安装必需）

GhostLock 只负责提权。KernelSU 的安装依赖 **ksud**（内含各 KMI 版本的 `kernelsu.ko`）：

| 获取方式 | 说明 |
|----------|------|
| **ReSukiSU APK**（推荐） | 安装 [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU)，APK 捆绑了 `libksud.so` |
| **CI 发行版** | 从 [ReSukiSU CI](https://github.com/cctv18/ReSukiSU_CI/releases) 下载 `ksud-aarch64-linux-android.zip` |

> 没有 ksud 时，利用仍可获得 uid=0 的 root shell，但 KernelSU 不会被安装，`su` 不会持久化。

---

## 构建方法

### 依赖

- Android NDK（r25+）
- 设置环境变量 `ANDROID_NDK_HOME` 或 `ANDROID_NDK_ROOT`

### 编译

```bash
# 默认编译 (API 35)
make

# 指定 API 级别
make API=34

# 指定 NDK 路径
NDK=/path/to/android-ndk make
```

### 产物

编译后生成 `ghostlock` — 静态链接的 ARM64 ELF 可执行文件。

---

## 使用方式

### 一次性设置

```bash
# 1. 启用 ADB TCP 模式
adb tcpip 5555

# 2. 推送 ADB 密钥（bootstrap 模式需要）
adb push ~/.android/adbkey /data/local/tmp/a/adbkey

# 3. 推送利用程序
adb push ghostlock /data/local/tmp/a/e
adb shell chmod 755 /data/local/tmp/a/e
```

> 首次成功后，`resetprop` 会自动设置 `persist.adb.tcp.port=5555`，后续重启可全自动运行。

---

## 运行模式

### 完整利用（ADB shell 环境）

```bash
/data/local/tmp/a/e
```

- perf 可用，能精确泄露子进程 `task_struct` 地址
- 两阶段写入：W1 关 SELinux → W2 提权 → 加载 KernelSU

### Bootstrap 模式（App 上下文，有 seccomp 限制）

```bash
/data/local/tmp/a/e --bootstrap
```

1. **Write 1** → 关闭 SELinux
2. 利用 SELinux 关闭后的权限执行 `setprop` 启用 ADB TCP 5555
3. 内置 Mini ADB 客户端通过 `127.0.0.1:5555` 回连
4. 使用预推送的 RSA 密钥完成 ADB 认证
5. 在 ADB shell（无 seccomp 限制）下执行完整利用

### 仅 Write 1（只关闭 SELinux）

```bash
/data/local/tmp/a/e --write1
```

- 最多尝试 20 次
- 可用于调试或需要临时关闭 SELinux 的场景

---

## 技术细节

### 1. 运行时内核匹配

偏移量存储在 `src/devices/offsets.h` 的查找表中，以 `uname -r` 为键。程序启动时自动匹配，未知内核直接拒绝运行。

```c
static const struct kernel_offsets known_offsets[] = {
  OFFSETS_ENTRY("6.12.38-android16-5-...-ab14275539-4k", ...),
  OFFSETS_ENTRY("6.12.38-android16-5-...-ab14552068-4k", ...),
  OFFSETS_ENTRY("6.12.23-android16-5-...-ab14541642-4k", ...),
  { .uname_r = NULL }  /* 哨兵 */
};
```

#### 偏移来源

| 类型 | 数量 | 提取方式 |
|------|------|----------|
| 全局符号偏移（kallsyms） | 28 | `tools/extract_target.py` |
| 结构体字段偏移（BTF） | 57 | `tools/extract_btf.py` |
| 推导值 | 9 | 自动计算 |
| 固定常量 | 12 | 无需提取 |

#### BTF 验证的结构体

| 结构体 | 字段数 | 用途 |
|--------|--------|------|
| `task_struct` | 17 | 进程描述符、cred、seccomp |
| `rt_mutex_waiter` | 6 | UAF 伪造结构体 |
| `cred` | 4 | 用户凭证、capabilities |
| `seccomp` | 3 | seccomp 过滤状态 |
| `pipe_inode_info` | 11 | 管道缓冲区操作 |
| `file_operations` | 13 | 伪文件操作表 |
| `mm_struct` | 1 | 内存描述符 owner |

### 2. KASLR 绕过

#### SLIDE 模式 — boot_id 泄露

当内核指针被限制读取时（`kptr_restrict`），利用 `boot_id` 被内核地址覆写的特性泄露基址：

```
读取 /proc/sys/kernel/random/boot_id
  └─ UUID 中被覆写为 nfulnl_logger 地址
      └─ 计算 KASLR slide = leaked_addr - image_offset
          └─ 得到 kaslr_base
```

#### FOPS/CFI 模式 — 文件操作表泄露

当有 ashmem 设备访问权限时，通过 configfs 读写原语读取 ashmem 的 fops 表，从中提取函数指针计算 KASLR 偏移：

```
打开 ashmem 设备
  └─ 读取 fops 表中的 open / ioctl / mmap 等函数指针
      └─ 与 image 基址偏移对比 → 得到 kaslr_base
```

### 3. 内核堆喷射

在 order-3（32KB）大页上布置伪造内核对象：

- **伪造 `file_operations` 表** — 劫持 ashmem 设备的 miscdevice fops 指针
- **伪造 `rt_mutex_waiter`** — 模拟 PI 链上的 waiter 节点
- **伪造 `task_struct`** — PI 链遍历中的 task 引用
- **伪造 `rt_mutex`（锁）** — 正确的锁等待者和所有者信息

堆喷射通过 **SKB（socket buffer）** + **KernelSnitch** 实现：
- **KernelSnitch** — 利用 futex 哈希碰撞泄露 `mm_struct` 地址，辅助堆风水
- **SKB 喷射** — 通过 `sendmsg` 填充内核堆

### 4. 物理内存读写（Pipe）

获取 KASLR base 后，利用管道缓冲区实现物理地址级别读写：

```
1. 定位管道缓冲区在 physmap 中的位置
2. 伪造 pipe_buffer 操作表指向已知的 pipe_buf_ops
3. 劫持 pipe_buffer 的 page 字段指向目标物理地址
4. 通过 pipe read/write 实现任意物理地址读写
```

支持操作：`pipe_read64`、`pipe_write64`、`pipe_phys_read_data`、`pipe_phys_write_data`

### 5. 两阶段写入

#### Write 1 — 关闭 SELinux

```
目标: selinux_state.enforcing (offset 0x00)
方式: child-node PI 写入 → 伪造 waiter 的 __rb_parent_color
      指向 selinux_enforcing - 8，rb-tree 重平衡时写入 0x00
```

SELinux 关闭后，`setprop` 等受 SELinux 限制的操作不再被阻止。

#### Write 2 — 提权到 root

```
目标: 子进程的 cred 指针
方式: 1. fork 子进程 → perf_find_task() 定位 task_struct
      2. 计算 cred 字段偏移
      3. child-node PI 写入 → cred = init_cred (uid=0, full caps)
      4. 清除 seccomp (TIF_SECCOMP + seccomp 结构体清零)
```

写完 cred 后还会进行 **capability 回读验证**，确保写入生效。

### 6. 内置 Mini ADB 客户端

`src/core/miniadb.c` — 用于 bootstrap 模式的精简 ADB 协议客户端：

```
1. TCP 连接 127.0.0.1:5555
2. A_CNXN → 连接请求 (version + max payload + host::
3. A_AUTH → 收到 RSA token 挑战
4. dlopen("libcrypto.so") → PEM_read_bio_RSAPrivateKey → RSA_sign
5. A_AUTH (AUTH_SIGNATURE) → 签名响应
6. A_CNXN → 连接建立
7. A_OPEN "shell:/data/local/tmp/a/e" → 执行完整利用
```

支持 SHA-1 和 SHA-256 两种签名算法。

---

## 文件结构

```
ghostlock-oneplus/
├── Makefile                        # 构建配置 (NDK 交叉编译)
├── README_CN.md                    # 说明文档
├── src/
│   ├── core/                       # 核心利用代码
│   │   ├── main.c                  # 入口：两阶段写入 + root shell
│   │   ├── fops.c                  # FOPS/CFI 模式：pselect 路由、PI 写入、KASLR 泄露
│   │   ├── util.c                  # 工具函数：堆喷射、KASLR 地址计算、内核 R/W 原语
│   │   ├── slide.c                 # SLIDE 模式：boot_id KASLR 泄露 + pselect 路由
│   │   ├── pipe.c                  # 管道缓冲区物理内存读写 (physrw)
│   │   ├── root.c                  # cred 覆盖、seccomp 清除、root 子进程管理
│   │   ├── miniadb.c              # 内置 ADB 客户端 (TCP + RSA 认证)
│   │   ├── common.h               # 全局宏、结构体定义、函数声明、常量
│   │   ├── target.h               # 目标设备内存布局 / 结构体偏移 / KASLR 参数
│   │   ├── offset.h               # 编译时 target config 桥接 (#include TARGET_CONFIG_H)
│   │   └── kernelsnitch/          # KernelSnitch — mm_struct 地址泄露
│   │       ├── kernelsnitch.h     # 核心算法：futex 哈希碰撞 + 暴力搜索
│   │       ├── futex_hash.h       # futex 哈希函数
│   │       ├── timeutils.h        # CPU 时间戳测量 (RDTSC)
│   │       └── utils.h            # 辅助宏 (pr_info / SYSCHK / ASSERT 等)
│   └── devices/                   # 设备偏移表
│       ├── offsets.h              # 聚合所有设备偏移 (查找表 + 哨兵)
│       ├── ace6t/offsets.h        # 一加 Ace 6T 偏移 (2 个内核版本)
│       └── op15/offsets.h         # 一加 15 偏移 (1 个内核版本)
└── tools/                         # 偏移提取工具链
    ├── extract_target.py          # 从 kallsyms 提取全局符号偏移 (28 个)
    └── extract_btf.py             # 从 BTF 提取结构体字段偏移 (57 个)
```

### 核心模块说明

| 模块 | 文件 | 职责 |
|------|------|------|
| **入口** | `main.c` | 命令行解析、W1/W2 调度、bootstrap 流程 |
| **PI 路由** | `fops.c` / `slide.c` | pselect/select 栈布局、futex PI 链操控、竞争触发 |
| **KASLR** | `util.c` / `fops.c` / `slide.c` | 双模式绕过：boot_id 泄露 + fops 表泄露 |
| **堆喷射** | `util.c` | order-3 大页分配、SKB 喷射、伪造对象布局 |
| **物理 R/W** | `pipe.c` | pipe_buffer 劫持、任意物理内存读写 |
| **提权** | `root.c` | cred 覆盖、capability 验证、seccomp 清除 |
| **ADB** | `miniadb.c` | 精简 ADB 协议客户端、RSA 认证 |
| **泄露引擎** | `kernelsnitch/` | futex 哈希碰撞定位 mm_struct |

---

## 适配新设备

只需目标设备的 `boot.img` 即可提取偏移——无需 root，无需设备访问。

### 提取偏移

```bash
# 1. 从 boot.img 提取内核
python -c "import struct; d=open('boot.img','rb').read(); \
           open('kernel','wb').write(d[4096:4096+struct.unpack_from('<I',d,8)[0]])"

# 2. 获取 kallsyms 符号表 (有 root: adb shell su -c 'cat /proc/kallsyms' > kallsyms.txt)
#    或从 vmlinux: nm vmlinux > kallsyms.txt

# 3. 提取全局符号偏移
python tools/extract_target.py     # 28 个偏移，自动交叉验证

# 4. 提取结构体字段偏移
python tools/extract_btf.py kernel  # 57 个偏移，自动验证
```

### 适配步骤

1. 在 `src/devices/` 下创建 `<设备名>/offsets.h`，参考 `ace6t/offsets.h`
2. 在 `src/devices/offsets.h` 中 `#include` 新文件
3. 如有必要，更新 `src/core/target.h` 中的 `KIMAGE_TEXT_BASE` 和内存布局
4. 重新编译

### 跨设备调整项

| 调整项 | 说明 |
|--------|------|
| `VA_BITS` | 48 vs 39 → 更新 `target.h` 内存布局 |
| 时序参数 | 调整 `common.h` 中的 `PSELECT_*` 参数 |
| ashmem 实现 | C vs Rust → 更新 extract_target.py 符号匹配规则 |
| secureguard | 无 OnePlus secureguard 可简化部署流程 |

---

## 常见问题

### Q: 运行后没有任何输出？

检查是否在 **ADB shell** 环境中运行。App 上下文应使用 `--bootstrap` 模式。

### Q: "no offsets for this kernel"？

你的内核版本尚未收录。按[适配新设备](#适配新设备)提取偏移后重新编译。

### Q: Write 1 老是失败？

- 确保固定到正确的 CPU 核心（`CORE` 宏）
- 调整时序参数 `PSELECT_ENTER_DELAY_USEC`
- 检查是否有其他进程干扰堆状态

### Q: Write 2 的 perf 返回 0？

- 检查 seccomp 是否阻止了 `perf_event_open`（App 上下文应使用 `--bootstrap`）
- 确认 KASLR 绕过已成功获取 `kaslr_base`

### Q: KernelSU 未加载？

- 确认 `ksud` 存在且可执行
- 检查网络策略文件是否完整（`load_policy` 修复）

---

## 许可

**仅限授权安全研究和教育用途。**