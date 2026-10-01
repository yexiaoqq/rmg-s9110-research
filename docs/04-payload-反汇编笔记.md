# 04 · payload 反汇编笔记

> 目标文件：`u-usr.so`（FZI1 payload，104128 字节）
> 方法：`objdump -d` 反汇编 + 字符串交叉引用（`refs.out`）。
> 本页只记录**可复现的硬证据**，不猜测。

## 1. 碰撞阈值判断点（核心突破）

### 函数

`kernelsnitch_find_collisions`，起始 VA `0xd014`，大小 `0x4d0`。

### 关键指令

```asm
d288:cmpx21, x23
d28c:b.ned2b0
```

- `x21` 是**阈值**，来源：`x21 = [x19, #24]`（即对象偏移 `+0x18` 字段）。
- `x23` 是**实际凑到的碰撞数**。
- 如果两者**不相等**，就跳到 `d2b0`，打印：

```text
only found %zd collisions -> cannot continue
```

### 阈值从哪来

阈值字段 `[obj+0x18]` 由 `kernelsnitch_setup`（VA `0xccd8`）写入：

```asm
cd58:strx22, [x19, #24]
```

而 `x22` 是 `kernelsnitch_setup` 的**第 3 个参数**（ARM64 调用约定里普通参数是 `x0,x1,x2,...`）。

> **结论：碰撞阈值不是硬编码在函数里的，而是由调用者传进来的。**
> 想改阈值，必须回到调用 `kernelsnitch_setup` 的地方（如 `prepare_kernel_page` 或其 wrapper ）改传入的第 3 个参数。

## 2. 扫描参数：硬编码，不可用环境变量覆盖

`prepare_kernel_page` 中，打印下面这行日志的调用位于 `0xee00`–`0xee08`：

```text
controlled mm group search scans=%d objects=%zu dma32_skip=%d
```

对应寄存器赋值（**都是立即数硬编码**）：

| 参数 | 寄存器 | 值 |
|---|---|---|
| scans | `w1` | `256`（`#0x100`） |
| objects | `w2` | `32`（`#0x20`） |
| dma32_skip | `w3` | `8`（`#0x8`） |

而且该函数内**无 `getenv` / `strtoul`**。

> **结论：`scans=256 / objects=32 / dma32_skip=8` 无法通过环境变量调整。**
> 要改它们只能改二进制，或找到其他覆盖路径（例如通过调用者传入）。

## 3. 隐蔵的诊断环境变量（重要）

在 `run_exploit`（VA `0xc634`）头部发现两个 `getenv`，在 `find_collisions` 里发现第三个：

| 环境变量 | 字符串位置 | 读取位置 | 作用（推测） |
|---|---|---|---|
| **`SLIDE_ONLY`** | `@0x8b74` | `run_exploit @0xc6c8` | 只跑 KASLR 泄露（slide），不进入后续 |
| **`P0_ONLY`** | `@0xa553` | `run_exploit @0xc6d8` | 只跑 P0 阶段 |
| **`KSNITCH_TIMING_DIAG`** | `@0x9bd1` | `find_collisions @0xd2c8` | 输出碰撞时序诊断 |

> **意义：这三个是 payload 内置的“分段开关”，可用于零风险的分阶段验证——不必每次都跑完整流程。**
> 这是重跑实验前必须利用的**信息侧杠杆**。

另外 `run_exploit` 内还有：
- `EXPLOIT_ATTEMPTS`（字符串 `@0x50b0`）
- `EXPLOIT_ATTEMPT_TIMEOUT_SEC`（字符串 `@0xa3ab`）
- 以及交接件记录的其他 env（`P0_ATTEMPT_TIMEOUT_SEC` / `SLIDE_P0_OFFSET` 等）。

## 4. payload 完整接口（从二进制抠出，非源码猜测）

### 环境变量

| 变量 | 说明 |
|---|---|
| `LD_PRELOAD` | 预加载 payload 本身 |
| `CVE43499_ROOT_HELPER` | helper 可执行文件路径 |
| `EXPLOIT_ATTEMPTS` | 总尝试次数 |
| `EXPLOIT_ATTEMPT_TIMEOUT_SEC` | 单次 attempt 超时 |
| `P0_ATTEMPT_TIMEOUT_SEC` | P0 阶段超时 |
| `SLIDE_P0_OFFSET` | 手动指定 slide/P0 偏移 |
| `SLIDE_ONLY` / `P0_ONLY` / `KSNITCH_TIMING_DIAG` | 诊断开关（本次新发现） |

子进程内部还会 `setenv`：`P0_GATE_PAGE_STRUCT` `P0_PROBE_PAGE_STRUCT` `S23_SUPERVISOR_ATTEMPT` `SLIDE_ENTER_DELAY_USEC`。

### helper 参数

`--daemon` `--late-load` `--package-name` `--run-payload` `--umh`

### 内嵌产物路径

- `/data/local/tmp/.rmg-trace-io-%d`
- `/data/local/tmp/temp_su.sock`

### App 在 Shizuku 模式下的发起命令

```text
/system/bin/sh -c true
```

附 env：

```text
EXPLOIT_ATTEMPTS=24
P0_ATTEMPT_TIMEOUT_SEC=45
EXPLOIT_ATTEMPT_TIMEOUT_SEC=120
CVE43499_ROOT_HELPER=/data/local/tmp/ksu-helper
LD_PRELOAD=/data/local/tmp/ksu-payload
```

## 5. futex 哈希路径（已排除的假说）

| 函数 | VA | 作用 |
|---|---|---|
| `futex_hash_no_trunc` | `0xcb54` | Jenkins Lookup3，initval `0xdeadbeff`，24 字节 key |
| `__futex_hash` | `0xcc1c` | 对上述结果 `& (hashsize-1)` |
| `futex_hash` | `0xcc64` | 带缓存的封装 |
| `futex_init` | `0xcc44` | 用 `sysconf(_SC_NPROCESSORS_ONLN)*256` 填 `futex_hashsize` |

**排除结论**：本机 `online=possible=present=0-7`、`nproc=8` → `8*256=2048`，与内核一致。
所以“hashsize 算错”不是卡点原因。

工作线程在 `d7fc`/`d874` 调 `futex_hash` 并与 `kernel_write_data` 结果 `cmp` 判等（哈希相等即判同桶）。

## 6. 小结

| 问题 | 结论 |
|---|---|
| 碰撞阈值在哪？ | 对象 `+0x18` 字段，由 `kernelsnitch_setup` 第 3 参传入 |
| 扫描参数 256/32/8 可改吗？ | 否，硬编码 |
| 有无可用诊断开关？ | **有**：`SLIDE_ONLY` / `P0_ONLY` / `KSNITCH_TIMING_DIAG` |
| 碰撞/时序路径有本地旋钮吗？ | 无（多轮受控实验 `delta_med` 全 0） |

> 因此，改动只能落到**上游 payload 源码层**（需向上游反映），而本地能做的最大价值是：
> 用 `SLIDE_ONLY` / `P0_ONLY` / `KSNITCH_TIMING_DIAG` 把**失败发生在哪个阶段**再缩小一次。
