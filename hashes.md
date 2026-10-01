# 哈希台账（Hash Ledger）

> 为什么要有这一页？
> 因为我们踩过一个真实的坑：手机里同时存在 **FZH3** 和 **FZI1** 两个固件的 payload，
> 一不留神就会拿错文件，跑几十分钟后才从日志里发现"跑错版本了"。
> 哈希是唯一不会说谎的身份证明。**每次实验前，先对哈希。**

## 1. 设备基线（每次实跑前后都核对）

| 项目 | 值 |
| --- | --- |
| 型号 | SM-S9110（代号 dm1q） |
| 固件版本 | S9110ZCS8FZI1 |
| Android | 16（SDK 36） |
| 内核 | 5.15.189-android13-8-3251900-abS9110ZCS8FZI1 |
| KMI | android13-5.15 |
| Bootloader | flash.locked=1（锁定） |
| 校验启动 | verifiedbootstate=green |
| Knox 熔断丝 | warranty_bit=0（未熔断） |

三条自查命令（只读，零风险）：

```sh
getprop ro.build.fingerprint       # 应含 S9110ZCS8FZI1
getprop ro.boot.flash.locked       # 应为 1
getprop ro.boot.verifiedbootstate  # 应为 green
```

## 2. Payload 二进制

| 文件 | 大小(B) | SHA-256 | 说明 |
| --- | --- | --- | --- |
| `cve-2026-43499-app.so` | 104128 | `f8a9560d76bd4d7f86d3f0822960a40fff31a7eb579a0bbe95ef37407ac3909f` | **本机 FZI1 正确 payload**，即 `u-usr.so` |
| `ksud-dm3q-S918BXXSAFZF5-kdp` | 4879560 | `5da5818d…`（见 journal 001） | helper（KernelSU daemon 相关） |
| `u-dm1q911u1.so` | 130168 | 未归档 | 早期变体，非本次使用 |
| `u-up.so` | 14 | — | 占位/空文件 |
| `yz.so` | 14 | — | 占位/空文件 |

> ⚠️ **易错点**：设备上历史残留的 `/data/local/tmp/ksu-payload` 是 **FZH3** 变体，
> 与本机 FZI1 不匹配。用它跑出来的失败日志里会出现
> `label=dm1q-S9110ZCS8FZH3-…` —— 看到这个 label 就说明拿错文件了。

## 3. 自建 feed / 仓库材料

| 文件 | 大小(B) | SHA-256 | 说明 |
| --- | --- | --- | --- |
| `targets-v3.json`（本地） | 600 | `eeb09b83…`（见 journal 001） | schemaVersion 3，payloadId=dm1q-S9110ZCS8FZI1 |
| `upstream-targets-v3.json` | — | `296438aab68ce9ab60928d77b58c3b180198d1ae4ad886874d9225a40dec48af` | 上游原始 feed 备份 |
| `feed.json` | 12242 | 同上 | 与 upstream 同哈希 |

## 4. 重打包产物

| 文件 | 大小(B) | SHA-256 | 说明 |
| --- | --- | --- | --- |
| `rmg-rebuilt.apk` | 68075761 | `4af329af47fe3d33075883b28e0cb0af2de9127ba5bdded21634ce86440b9c64` | 6 处 URL 等长替换 + apksigner 重签 |
| `cur-base.apk` | 68075761 | `4af329af47fe3d33075883b28e0cb0af2de9127ba5bdded21634ce86440b9c64` | **与上一行完全相同** |

> 🔎 **重要发现**：`rmg-rebuilt.apk` 与 `cur-base.apk` 的 SHA-256 **完全一致**，
> 说明当前设备上跑的"重打包版"和"基线备份"其实是同一个文件。
> 留档这条，是为了以后能追溯"当时到底跑的是哪一份 APK"。

## 5. 文档留档快照

> 每次 push 到 GitHub 后，回填本次提交的 commit SHA，形成前后呼应。

**首次上线提交**：`80d7bea7a5d24ebe3d75b79ed3ec6eff06c40d28`（2026-10-02，19 文件全量）
**上线前基线**：`9884d754141d6cae252944fcf3664fe61354a156`（仓库创建时初始 commit）

| 文件 | 大小(B) | 备注 |
| --- | --- | --- |
| README.md | 5451 | |
| DISCLAIMER.md | 1940 | |
| hashes.md | 3978 | 本页 |
| docs/00-名词解释.md | 4943 | |
| docs/01-项目背景.md | 2709 | |
| docs/02-目标设备与环境.md | 2159 | |
| docs/03-当前进展与卡点.md | 3850 | |
| docs/04-payload-反汇编笔记.md | 5197 | |
| docs/05-上游PR调研笔记.md | 3059 | |
| analysis/README.md | 1676 | |
| analysis/01-反汇编环境与方法.md | 2842 | |
| analysis/02-碰撞阈值定位.md | 4021 | |
| analysis/03-futex哈希路径排除.md | 2986 | |
| analysis/04-上游5.15参数档对比.md | 2704 | |
| journal/README.md | 1950 | |
| journal/TEMPLATE.md | 804 | |
| journal/2026-10-02-001-*.md | 1545 | |
| journal/2026-10-02-002-*.md | 2225 | |
| journal/2026-10-02-003-*.md | 2845 | |
| journal/2026-10-02-004-*.md | — | 本次上线记录（随第二次提交入库） |
| rmg-handoff.md（设备侧） | 15525 | `68d21b1ec6ea1ee1564ea60634b4c2ac0f325fc36647c831563edc128a2a4b68` |

> 核验结果：远端 `git/trees/main?recursive=1` 与本地逐文件比对，**19/19 大小一致（ALL GREEN）**。
> 台账纪律：新增文件后同步本表；commit SHA 与文件清单成对出现，便于追溯"哪一版长什么样"。

## 6. 校验方法

Linux / macOS：

```sh
sha256sum <文件>
```

Windows PowerShell：

```powershell
Get-FileHash -Algorithm SHA256 <文件>
```

## 7. 记账纪律

1. **新增任何 payload / APK，先测哈希再写进本表**，不要等出问题时才回头找。
2. 哈希对不上 = 停止操作，先查是不是拿错固件版本。
3. 本表只增不删；作废条目保留并标注 `[作废]` 与原因。
