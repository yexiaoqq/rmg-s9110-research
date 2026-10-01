# 03 · futex 哈希路径排除

- **证据文件**：`cve-2026-43499-app.so`、`fp2.c`/`fp3.c`/`fp5.c`/`futexprobe.c`、探针实测结果
- **工具**：objdump + 沙箱交叉编译的 AArch64 用户态探针
- **关键地址**：`0xcb54`、`0xcc1c`、`0xcc64`、`0xcc44`

## 1. 假设

一个很自然的猜想：“KernelSnitch 靠 futex 哈希桶碰撞来区分同桶/异桶，
那是不是我们的内核 `futex_hashsize` 算出来跟 payload 预期的不一致，
导致碰撞统计永远为 0？”

## 2. 反汇编事实

| 符号 | VA | 内容 |
| --- | --- | --- |
| `futex_hash_no_trunc` | `0xcb54` | Jenkins Lookup3，initval `0xdeadbeff`，key 长 24 字节 |
| `__futex_hash` | `0xcc1c` | 对上述结果 `& (hashsize-1)` |
| `futex_hash` | `0xcc64` | 带缓存的封装 |
| `futex_init` | `0xcc44` | 用 `sysconf(_SC_NPROCESSORS_ONLN) * 256` 填全局 `futex_hashsize` @ `0x1a4b0` |

工作线程在 `d7fc/d874` 调用 `futex_hash`，并与 `kernel_write_data` 的结果 **cmp 判等**
（哈希相等即判为同桶）。

## 3. 数值核对（事实）

本机：

```
online  = 0-7
possible= 0-7
present = 0-7
nproc   = 8
```

payload 计算：`8 * 256 = 2048`。
内核侧 futex hashsize 亦为 **2048**。

**结论**：两边算出来的桶数**一致**，“hashsize 算错”这个假设**排除**。

## 4. 运行时探针（事实）

为了验证“同桶/异桶在哈希时是否可测”，在沙箱交叉编译了静态 AArch64 探针上机跑：

| 探针 | 做法 | 结果 |
| --- | --- | --- |
| `fp` | 首次 SAME vs OTHER 计时 | min 5→8 (+60%) / OTHER 7，**复测反向，不可复现** |
| `fp2` | 逐点扫描命中率 | 736/1023 → 864/1023，单调升 —— 属 DVFS/预热**漂移假象** |
| `fp3` | 配对交替测所有候选（含目标） | `delta_med = 0` |
| `fp5` | 4 线程、3.28M 次锤击、配对测 | `delta_med` 全 0，延迟恒定 8 ticks |

另有一次 App 单次诊断：

```
ksnitch timing diag approx_time=14 threshold=140 min=14 max=279 mean=14 n=32766
```

即 32768 个 slot **全落在 14 tick**，平坦、无区分度。

## 5. 结论与限定语

**结论**：在 **FUTEX_WAKE 短临界区**下，同桶/异桶在我们这台机器上
**没有可测的时间差异**，无法把碰撞筛选出来。

**限定语（重要）**：
- 这只否定“**当前短临界区测法**”，**不等于** futex 路径整体无效。
- 尚未测试的原语包括：带**真实等待者**的 `FUTEX_WAKE`、
  `FUTEX_REQUEUE`、以及 `futex_wait` 的**阻塞路径**（长临界区）。
- 本机在这条路径上**没有任何可用的本地运行时旋钮**，
  改动只能落到**上游 payload 源码层**（例如改进碰撞判定/时序测量方式）。

## 6. 与上游支持矩阵的关系

上游支持矩阵中 `android13-5.15 = Untested`，
与本机“FZI1 payload 走不通”的实测一致。
但两者都不足以推出“5.15 不可行”，只说明**当前 payload 的当前策略不适配**。