# feat_res 主频设置逻辑梳理

基于 `feat_res_mgnt`、`dfs` 模块及实际日志整理，便于后续对照代码学习。

---

## 1. 架构概览

```
业务模块 (音乐/通话/BT/Stream...)
        │
        ├─ wq_feat_res_start/stop()          ── 动态 MIPS 累加
        └─ wq_feat_res_fixed_start/stop()    ── 固定档位，不参与 MIPS 累加
                    │
                    ▼
           feat_res_mgnt (ACore 汇总)
                    │
        ├─ dfs_request() / dfs_release()     ── ACore/BCore
        └─ IPC → DCore                        ── DCore 同步
                    │
                    ▼
              DFS Server (dfs.c)
                    │
        dfs_freq_get_most_suitable()          ── 合法频率组合表
        dfs_freq_get_turbo() [可选]           ── 短时 Turbo 升频
                    │
                    ▼
           wq_clock_set_cores_clock()        ── 实际写硬件
                    │
                    ▼
           [DFS] set to A/B/D (MHz)          ── 最终生效频率
```

**三核频率格式**：`A/B/D` = ACore / BCore / DCore（单位 MHz）。AB 核通常同频。

---

## 2. 两种主频请求方式

### 2.1 动态请求（Dynamic）

| 项目 | 说明 |
|------|------|
| API | `wq_feat_res_start()` / `wq_feat_res_stop()` |
| 机制 | 各功能按 `g_feat_user_param[]` 贡献 MIPS，ACore 累加后换算标准频率 |
| 适用 | 音乐解码、ANC、VAD、录音等长期运行的功能 |

**MIPS 累加公式**（`wq_feat_res_dfs_hdl`）：

```
a_mips_sum = 3 + Σ(各活跃 user 的 acore mips) + 10% margin
b_mips_sum = 3 + Σ(bcore mips) + 10% margin
d_mips_sum = 3 + Σ(dcore mips) + 10% margin

a_new_freq = max(a_mips_sum, b_mips_sum) → 向上取标准档
d_new_freq = max(d_mips_sum, a_mips_sum) → 向上取标准档，且需为 a 的整数倍
```

- 固定底噪 `3`：`FEAT_RES_MIPS_FIXED_MARGIN`
- 空闲默认：`16/16` MHz（`FEAT_RES_IDLE_DFS_*`）
- ACore 负责汇总；DCore 频率经 IPC 下发

### 2.2 固定请求（Fixed）

| 项目 | 说明 |
|------|------|
| API | `wq_feat_res_fixed_start()` / `wq_feat_res_fixed_stop()` |
| 机制 | 直接 `dfs_request(固定频率)`，**不参与** MIPS 累加 |
| 适用 | 突发/临时场景：OTA、BT 连接、Stream 创建/销毁等 |
| timeout | `timeout > 0` 为临时提频，到期自动 `dfs_release` |

**固定档位表**（`g_feat_fixed_user_param[]`）节选：

| 枚举值 | 名称 | A/B/D (MHz) |
|--------|------|-------------|
| 4 | `FEAT_RES_FIXED_64_64` | 64/64/64 |
| 5 | `FEAT_RES_FIXED_64_128` | 64/64/128 |
| 15 | `FEAT_RES_FIXED_BLE_CON` | 32/32/32 |
| 18 | `FEAT_RES_FIXED_BT_SV` | 32/32/32 |
| 22 | `FEAT_RES_FIXED_STREAM_64_128` | 64/64/128 |

---

## 3. 日志格式说明

### 3.1 `[feat_res]` 动态日志

```
[feat_res] en:0 usr:0x0000100000000000 msk:0x0000000000000008 mod:0 exp:7/11/13 use:16/16->16/16
```

| 字段 | 含义 |
|------|------|
| `en` | 1=start，0=stop |
| `usr` | 本次操作的 user bit mask |
| `msk` | 操作后仍活跃的 user mask |
| `mod` | 0=`FEAT_RES_MODE_DEFAULT`，1=`FEAT_RES_MODE_DUAL_CH` |
| `exp` | 累加后的 a/b/d MIPS（标准化前） |
| `use` | `a_old/d_old -> a_new/d_new`（MHz） |

**user bit 与枚举**（`WQ_FEAT_RES_USR_E`，节选）：

| Bit | 枚举 | A/B/D MIPS |
|-----|------|------------|
| 3 | `WQ_FEAT_RES_USR_MUSIC_COMMON` | 4/7/9 |
| 5 | `WQ_FEAT_RES_USR_MUSIC_LDAC` | 39/46/45 |
| 18 | `WQ_FEAT_RES_USR_ANC_ADAPT_ANC` | 3/3/15 |
| 48 | `WQ_FEAT_RES_USR_VOL_CHANGE` | 30/30/30 |
| 49 | `WQ_FEAT_RES_USR_MUSIC_L2HC` | 16/32/128 |

### 3.2 `[feat_res]` 固定日志

```
[feat_res] en:0 fixed usr:22 exp:64/128 timeout:0
```

| 字段 | 含义 |
|------|------|
| `en` | 1=start，0=stop |
| `usr` | `FEAT_RES_FIXED_E` 枚举值（**不是** `WQ_FEAT_RES_USR_E`） |
| `exp` | acore_freq / dcore_freq（MHz） |
| `timeout` | 0=永久，>0=临时 ms |

### 3.3 `[DFS]` 日志

| 日志 | 含义 |
|------|------|
| `[DFS] set to 64/64/128` | 最终写入硬件的三核频率 |
| `[DFS] turbo set to 32/32/32` | Turbo 短时升频（`CONFIG_DFS_TURBO_ENABLE`） |
| `[DFS] turbo release to ...` | Turbo 定时结束，回落到合适频率 |

**Turbo 规则**（`hornet/dfs_freq.c`）：当动态算出的组合为 `16/16/D` 时，可短时升到更高 AB 档，例如：

- `16/16/32` → Turbo `32/32/32`
- `16/16/64` → Turbo `64/64/64`
- `16/16/128` → Turbo `64/64/128`

---

## 4. 示例日志逐条解析

以下假设日志为一次**功能退出 / 降频**过程（典型 teardown 顺序）。

### 4.1 固定档位释放

```
[feat_res] en:0 fixed usr:22 exp:64/128 timeout:0
```

| 项 | 值 |
|----|-----|
| 操作 | stop |
| user | 22 = `FEAT_RES_FIXED_STREAM_64_128` |
| 来源 | Stream 创建/销毁（`stream_utils.c`） |
| 预期 | ACore 64MHz，DCore 128MHz |

```
[feat_res] en:0 fixed usr:18 exp:32/32 timeout:0
```

| 项 | 值 |
|----|-----|
| 操作 | stop |
| user | 18 = `FEAT_RES_FIXED_BT_SV` |
| 来源 | BT Sniff/Active 提频（`bt_dfs_mgr.c`） |
| 预期 | 三核 32MHz |

### 4.2 DFS 实际频率变化

```
[DFS] set to 64/64/128    ← 曾有 STREAM_64_128 等 fixed 请求
[DFS] set to 64/64/64     ← 可能 64_64 fixed 或 turbo(16/16/64)
[DFS] set to 32/32/32     ← fixed 释放后，当前最高请求为 32M
[DFS] turbo set to 32/32/32  ← 目标为 16/16/32 时触发 turbo
```

DFS 取各核 `dfs_request` 的**最大值**，再查 `freq_table` 得到合法组合（如 `64/64/128`、`32/32/32`）。

### 4.3 动态 user 释放

```
[feat_res] en:0 usr:0x0000100000000000 msk:0x0000000000000008 mod:0 exp:7/11/13 use:16/16->16/16
```

| 项 | 解析 |
|----|------|
| stop bit 48 | `WQ_FEAT_RES_USR_VOL_CHANGE` |
| 剩余 msk `0x8` | bit 3 = `MUSIC_COMMON` |
| exp 7/11/13 | 3 + MUSIC(4/7/9) + margin ≈ 7/11/13 |
| use 16→16 | MIPS 仍映射到 16M，**不变频** |

```
[feat_res] en:0 usr:0x00020000000007f8 msk:0x0000000000000000 mod:0 exp:3/3/3 use:16/16->16/16
```

| 项 | 解析 |
|----|------|
| stop `0x7f8` | bit 3~9：MUSIC_COMMON、SBC_48K、LDAC、LHDC 等 |
| stop `0x200000000000` | bit 49：`MUSIC_L2HC` |
| 剩余 msk 0 | 无动态 user |
| exp 3/3/3 | 仅 margin，无业务 MIPS |
| use 16→16 | 动态路径仍为 16M |

---

## 5. 完整调用链

### Fixed 为例

```
wq_feat_res_fixed_stop(FEAT_RES_FIXED_STREAM_64_128)
  ├─ wq_feat_res_fixed_dfs_hdl(en=0)     // ACore
  │    └─ dfs_release() → DFS Server 重算
  │         └─ [DFS] set to x/x/x
  └─ ipc_srv_commit(DCore)               // 同步 DCore release
       └─ [feat_res] en:0 fixed usr:22 ...
```

### Dynamic 为例

```
wq_feat_res_stop(WQ_FEAT_RES_USR_MUSIC_COMMON)
  └─ share_task → wq_feat_res_dfs_hdl(en=0)
       ├─ 更新 user_mask，重算 MIPS
       ├─ [feat_res] en:0 usr:... msk:... exp:... use:...
       ├─ dfs_request/release (ACore，若频率变)
       └─ IPC 通知 DCore（若 dcore 频率变）
```

---

## 6. Fixed vs Dynamic 优先级

```
实际运行频率 = max(所有 dfs_request 持有的频率)
              ↑
    ├─ feat_res 动态 path 的 dfs_object
    └─ feat_res 固定 path 的 dfs_object（可多个）
```

- Fixed 与 Dynamic **共用** DFS 对象链表，取最高频
- 动态算 16M 但 Fixed 仍 hold 32M → 实际 **32M**
- 全部 release 后回落 **16M**（`WQ_FREQ_16M`）

---

## 7. 常见场景对照

| 场景 | 接口 | 典型频率 |
|------|------|----------|
| 音乐播放 | `wq_feat_res_start(MUSIC_*)` | 动态累加，可达 64/128+ |
| AANC Stream 创建 | `FEAT_RES_FIXED_STREAM_64_128` | 64/64/128 |
| BT LE 连接 | `FEAT_RES_FIXED_BLE_CON` | 32/32/32 |
| BT Active/Sniff | `FEAT_RES_FIXED_BT_SV` | 32/32/32 |
| LE Audio CIS 通话 | `FEAT_RES_FIXED_64_128` | 64/64/128 |
| 空闲 | 无请求 | 16/16/16 |

---

## 8. 关键源文件

| 文件 | 作用 |
|------|------|
| `wq-adk/components/feat_res_mgnt/cores/feat_res_mgnt.c` | MIPS 表、动态/固定调度、日志打印 |
| `wq-adk/components/feat_res_mgnt/inc/feat_res_mgnt.h` | user / fixed 枚举 |
| `wqcore/components/dfs/src/dfs.c` | DFS 请求/释放、Turbo、`[DFS]` 日志 |
| `wqcore/components/dfs/src/hornet/dfs_freq.c` | 合法频率组合表、Turbo 映射 |
| `wq-adk/components/bt_service/common/bt_dfs_mgr.c` | BT 相关 fixed 提频 |
| `wq-adk/components/stream_mgnt/src/stream_utils.c` | Stream 创建/销毁 fixed 提频 |

---

## 9. 调试技巧

1. **先看 `[DFS] set to`**：这是硬件最终频率。
2. **再看 `[feat_res]`**：区分 fixed（`fixed usr:N`）与 dynamic（`usr/msk/exp`）。
3. **对照枚举**：`fixed usr:22` → `FEAT_RES_FIXED_STREAM_64_128`；`msk:0x8` → bit 3 → `MUSIC_COMMON`。
4. **Turbo**：出现 `turbo set` 表示从 `16/16/x` 短时升 AB 核；定时结束会 `turbo release`。
5. **`use:16/16->16/16`**：MIPS 变化但未跨标准档，不会触发 `dfs_request`。

---

## 10. 日志时序还原（推测）

```
[高负载] STREAM_64_128 / 音乐 / BT 提频
    ↓
[DFS] set to 64/64/128 或 64/64/64
    ↓
Stream 销毁 → fixed usr:22 stop
    ↓
[DFS] set to 32/32/32（BT_SV 等仍 hold 32M）
    ↓
音乐/VOL/L2HC 等 dynamic stop → msk 归零，exp 3/3/3
    ↓
BT_SV stop → fixed usr:18 stop
    ↓
[DFS] turbo set to 32/32/32（16/16/32 目标 + Turbo）
    ↓
最终回落空闲或低功耗档
```

---

## 11. 专题：`fixed usr:18` 与暂停音乐

### 11.1 常见误解：`usr:18` ≠ `ANC_ADAPT_ANC`

```
[feat_res] en:0 fixed usr:18 exp:32/32 timeout:0
```

这条日志来自 **Fixed 固定提频** 路径，使用 `FEAT_RES_FIXED_E` 枚举，**不是** `WQ_FEAT_RES_USR_E`。

两套枚举里恰好都有数值 `18`，但含义完全不同：

| 枚举类型 | 值 18 对应 | 日志里会不会出现 |
|---------|-----------|----------------|
| `FEAT_RES_FIXED_E`（fixed 日志） | **`FEAT_RES_FIXED_BT_SV`** | ✅ 就是这条 |
| `WQ_FEAT_RES_USR_E`（dynamic 日志） | `WQ_FEAT_RES_USR_ANC_ADAPT_ANC` | ❌ 只会出现在 `usr:0x... msk:...` 那种日志里 |

`FEAT_RES_FIXED_BT_SV` 从 0 数下来是第 18 个：

```
0:16_16  1:32_32  ...  15:BLE_CON  16:BT_CIS_CON  17:BT_BIS_CON  18:BT_SV  19:BT_SV_HIGH_SPD
```

对应频率配置是三核 **32/32/32 MHz**：

```c
[FEAT_RES_FIXED_BT_SV] = FEAT_RES_FREQ(32, 32, 32),
```

`ANC_ADAPT_ANC` 如果还在跑，走的是 **dynamic** 路径，日志格式为：

```
[feat_res] en:0 usr:0x00040000... msk:... mod:0 exp:... use:...
```

其中 `usr` 的 bit 18 才对应 `ANC_ADAPT_ANC`（`BIT_64(18)`）。

### 11.2 这条日志逐字段含义

| 字段 | 含义 |
|------|------|
| `en:0` | **释放**固定提频（`wq_feat_res_fixed_stop`） |
| `fixed usr:18` | `FEAT_RES_FIXED_BT_SV`，BT 业务提频 |
| `exp:32/32` | ACore 目标 32MHz，DCore 目标 32MHz（B 核与 A 同频） |
| `timeout:0` | 永久 hold，不是临时提频 |

### 11.3 暂停音乐后为什么会出现这条 log？

`FEAT_RES_FIXED_BT_SV` 的启停由 `bt_dfs_mgr.c` 管理，和音乐状态、ACL 链路模式强相关。

**开始播歌时会 stop BT_SV（en:0）**

```c
void bt_dfs_notify_music_playing(void)
{
    if (dfs_mode & BT_DFS_DOWN_FREQ_SNIFF && BT_DFS_IS_START(BT_DFS_MODE_SNIFF)) {
        BT_DFS_STOP(FEAT_RES_FIXED_BT_SV, BT_DFS_MODE_SNIFF);
    }
}
```

播歌时音乐自己会拉高主频，BT 的 32M Fixed 提频会被关掉 → 打出 **`en:0 fixed usr:18`**。

**暂停音乐时通常会 start BT_SV（en:1），不是 stop**

```c
void bt_dfs_notify_music_stop(void)
{
    if (dfs_mode & BT_DFS_UP_FREQ_SNIFF && BT_DFS_IS_STOP(BT_DFS_MODE_SNIFF)) {
        BT_DFS_START(FEAT_RES_FIXED_BT_SV, BT_DFS_MODE_SNIFF);
    }
}
```

前提：**主从链路处于 Active 模式**，且 PLINK/TLINK 已连接。如果已经是 Sniff 模式，`music_stop` **不会**启动 BT_SV。

**暂停后若链路回到 Sniff，会再次 stop BT_SV（en:0）**

```c
if (dfs_mode & BT_DFS_UP_FREQ_SNIFF) {
    BT_DFS_START(FEAT_RES_FIXED_BT_SV, ...);
} else if (dfs_mode & BT_DFS_DOWN_FREQ_SNIFF) {
    BT_DFS_STOP(FEAT_RES_FIXED_BT_SV, ...);   // → en:0 fixed usr:18
}
```

暂停后看到 `en:0 usr:18`，更可能是：

1. **暂停后 ACL 进入 Sniff**，`bt_dfs_notify_br_mode_changed` 把 BT_SV 关掉；或
2. 日志里混了**开始播歌时**那次 `music_playing` 的 stop（时间上和暂停挨得近，容易误判）

### 11.4 为什么不是 16/16/16，而是 `turbo set to 32/32/32`？

**16/16/16 是"完全空闲"档**，需要同时满足：

- 所有 Fixed `dfs_request` 都 release
- 所有 Dynamic `feat_res` user 都 stop，或 MIPS 累加后仍落在 16M 档

耳机暂停音乐、BT 仍连接、ANC 仍开时，通常**达不到**这个条件。

**Turbo 32/32/32 的含义**

```c
/* 16/16/32 -> 32/32/32 */
if ((clks[WQ_CORES_ACORE] == 16) && clks[WQ_CORES_DCORE] == 32) {
    turbo_clks[WQ_CORES_ACORE] = 32;
    ...
}
```

说明合并后的目标本来是 **16/16/32**（A/B 16M，D 32M），Turbo 把 A/B 也短时拉到 32M。

**暂停后 DCore 为何还要 32M？**

| 来源 | 说明 |
|------|------|
| **ANC 等算法（Dynamic）** | `ANC_ADAPT_ANC` mips=3/3/15，还有 `NOISE_DET`、`WIND_DET` 等，DCore MIPS 累加后常 >16M |
| **AB 核 16M、D 核 32M 的组合** | 代码会强制把 A 核抬高，避免效率过低 |
| **BT Fixed 仍 hold** | `FEAT_RES_FIXED_BLE_CON`（15）也是 32/32/32，LE 连着就会维持 32M |
| **Turbo 机制** | 16/16/32 → 短时升到 32/32/32，定时结束后再回落 |

`ANC_ADAPT_ANC` 用的是 **dynamic** `wq_feat_res_start/stop`，**不会**打出 `fixed usr:18` 这种日志；它会让 DCore 继续要 MIPS，从而阻止落到 16/16/16。

### 11.5 暂停音乐后的典型频率变化

```
播歌中
  ├─ Dynamic: MUSIC_COMMON / LDAC / L2HC 等（高 MIPS）
  ├─ Fixed BT_SV 通常被 music_playing 关掉（en:0 usr:18）
  └─ DFS 可能是 64/64/128 等高档

暂停
  ├─ Dynamic: 音乐 user 逐个 stop → MIPS 下降
  ├─ Stream destroy: fixed usr:22 (STREAM_64_128) 短暂 start/stop
  ├─ music_stop: 若 Active 链路 → 可能 start BT_SV (en:1 usr:18)
  ├─ 若进 Sniff → 可能 stop BT_SV (en:0 usr:18)
  └─ ANC/VAD/BT 等仍活跃 → 合并后 16/16/32 → Turbo → 32/32/32

真正 16/16/16
  └─ 需 BT 断开 + ANC/算法全停 + 无 Fixed hold（盒内待机级别）
```

### 11.6 小结

1. **`fixed usr:18` = `FEAT_RES_FIXED_BT_SV`（BT 32M 固定提频）**，不是 `ANC_ADAPT_ANC`。
2. **`en:0` = 释放**该 Fixed 提频；暂停后若链路进 Sniff，出现这条 log 是合理行为。
3. **暂停 ≠ 空闲**：ANC、BT 连接、LE 等仍会让主频停在 32M 档；`turbo set to 32/32/32` 说明系统认为 DCore 仍需 32M，Turbo 把 A/B 也同步拉高。
4. **`16/16/16` 只在几乎无业务时才会出现**，不是"没播歌"的默认状态。

---

## 12. 实际日志案例分析（暂停音乐完整流程）

以下基于 2026-06-05 15:15:53 ~ 15:16:12 的完整抓包 log，验证前述理论。

### 12.1 完整时间线（按因果）

```
15:15:53.643  ──► 音乐真正停止（暂停）
       │
       │  ~6 秒（ACL 仍 Active，BT 维持较高调度）
       ▼
15:15:57.634  ──► Link1 进 Sniff（BT_DFS res=0，暂不降频）
       │
       │  ~1.7 秒
       ▼
15:15:59.362  ──► Link0 也进 Sniff → 全链路 Sniff → 释放 BT_SV(usr:18)
       │              16/16/32 → turbo 32/32/32 → 16/16/16
       │
       │  ~1 秒（Turbo 定时器）
       ▼
15:16:00.330  ──► turbo release → 稳态 16/16/16
       │
       ▼
15:16:00+     ──► 三核进深睡，traffic=0，stream_music=0%
```

### 12.2 阶段 1：音乐暂停（15:15:53.643）

这是**真正的暂停点**，比后面的 DFS 日志早约 6 秒：

| 时间戳 (ms) | 关键 log | 含义 |
|-------------|----------|------|
| 321297 | `[asrc] music underrun 3` | 音乐数据断流（暂停前兆） |
| 321302 | `[audio_sys] playbk_stop` | 停止播放 |
| 321304 | `[DAC][close]` | DAC 关闭 |
| 321307 | `[stream] destory id 1` | 销毁 music stream |
| 321307 | `[music] cmd:4 state:3->0` | 音乐状态机 → 停止 |
| 321308 | `[DAC] pa off` | 功放关闭 |

此时：

- DCore `stream_music` 仍有 32% CPU（15:15:55），DSP 侧还在收尾
- BT `traffic = 10.9kb/s`（15:15:55），ACL 链路仍活跃，**尚未进 Sniff**

**结论**：暂停瞬间不会立刻到 16/16/16，因为 BT 链路还在 Active/Sniff 过渡中。

### 12.3 阶段 2：两条 ACL 先后进 Sniff（15:15:57 ~ 15:15:59）

#### Link1 先进 Sniff（15:15:57.634）

```
[SNIFF ENTER]link:1 sniff intv:384
[BT_DFS] action=2 res=0 state 0 1
```

- `res=0`：Link0 可能仍 Active，**还不能**关 `BT_SV`
- `state 0 1`：Sniff 提频状态已为 1（之前 Active 时开过 `FEAT_RES_FIXED_BT_SV`）

#### Link0 也进 Sniff（15:15:59.362）—— 触发降频

```
[SNIFF ENTER]link:0 sniff intv:386
[BT_DFS] action=2 res=8 state 0 1          ← res=8 = DOWN_FREQ_SNIFF
[DFS] core1 send request 16
[feat_res] en:0 fixed usr:18 exp:32/32     ← 释放 BT_SV
[DFS] request clk 16/16/32
[DFS] turbo set to 32/32/32
[DFS] core2 send request 16
[DFS] request clk 16/16/16
```

**因果链**：

```
RM/CM 全部 Sniff
    → bt_dfs: DOWN_FREQ_SNIFF (res=8)
    → stop FEAT_RES_FIXED_BT_SV (usr:18, 32M)
    → BCore 先 release → 16/16/32
    → Turbo 短时升 AB 到 32M
    → DCore 也 release → 16/16/16
```

`usr:18` 是 **`FEAT_RES_FIXED_BT_SV`**，由 **全链路 Sniff** 触发，不是音乐暂停 API 直接触发。

### 12.4 阶段 3：Turbo 结束，稳态 16/16/16（15:16:00.330）

```
[DFS] turbo release to 16/16/16
```

| 时间 (ms) | 事件 | 间隔 |
|-----------|------|------|
| 326988 | turbo set to 32/32/32 | — |
| 327988 | turbo release to 16/16/16 | **≈1000ms** |

与 `CONFIG_DFS_TURBO_DELAY` 一致：Turbo 只是 **约 1 秒的过渡**，不是稳态频率。

### 12.5 阶段 4：暂停后稳态（15:16:00 之后）

| 指标 | 暂停前 (15:15:55) | 暂停后 (15:16:00+) |
|------|-------------------|-------------------|
| BT traffic | 10.9 kb/s | **0.0 kb/s** |
| DCore stream_music | 32% CPU | **0% CPU** |
| DCore 深睡 | ds 1020-20% | **ds 4966-99%** |
| ACore IDLE | 87% | **96%** |
| BCore IDLE | 80% | **95~98%** |
| DFS 频率 | 高（播歌） | **16/16/16** |

系统已进入典型的**暂停后低功耗稳态**。

### 12.6 疑问与 log 对照结论

| 疑问 | 完整 log 的结论 |
|------|----------------|
| `fixed usr:18` 是 ANC 吗？ | **否**，是 `FEAT_RES_FIXED_BT_SV`（BT 32M 固定提频） |
| 是暂停直接触发的吗？ | **否**，暂停在 15:15:53；usr:18 在 15:15:59，由**全链路 Sniff** 触发 |
| 暂停后应该是 16/16/16 吗？ | **是**，最终在 15:16:00 落到 16/16/16，只是有 ~6 秒延迟 |
| turbo 32/32/32 是稳态吗？ | **否**，仅持续约 1 秒，随后 `turbo release` |

### 12.7 为何暂停到 16/16/16 有 ~6 秒延迟？

```
15:15:53  音乐停 → DAC/Stream 销毁
    ↓     ACL 仍 Active，BT_SV(32M) 可能仍 hold
    ↓     手机/协议栈协商进 Sniff（非即时）
15:15:57  Link1 → Sniff（Link0 仍 Active，不降频）
15:15:59  Link0 → Sniff（全 Sniff → 释放 BT_SV）
15:16:00  稳态 16/16/16
```

延迟来自 **BT 链路模式切换**，不是 feat_res 算频慢。

### 12.8 关于 log 里未出现的 feat_res 动态日志

暂停时（15:15:53）应有 `wq_feat_res_stop(MUSIC_*)` 等动态释放，但这份 log **没有出现** `[feat_res] en:0 usr:0x...` 行。可能原因：

1. 动态 feat_res 日志级别未开（VERBOSE/INFO 过滤）
2. Stream destroy 的 `fixed usr:22` 提频时间极短，未被采样到
3. 音乐 MIPS 释放后 `use:16→16` 未变档，未触发 DFS 打印

这不影响结论：**最终频率由 Sniff 后 BT_SV 释放 + 三核 request 16 决定**。

### 12.9 案例一句话总结

> **15:15:53 音乐暂停 → 约 6 秒后两条 ACL 全进 Sniff → 释放 BT_SV(usr:18) → 中间 turbo 32/32/32 约 1 秒 → 15:16:00 稳态 16/16/16。**

`en:0 fixed usr:18` 和 `turbo set to 32/32/32` 是**暂停后 BT 进低功耗 Sniff 的正常降频过程**，不是 ANC 相关，也不是异常。

---

*文档生成自 feat_res / DFS 代码分析与实际日志对照。*
