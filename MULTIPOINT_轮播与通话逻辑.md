# 一拖二：轮播逻辑与通话场景分析

> **术语说明**：一拖二双设备音乐切换采用的是 **轮播（Carousel）** 逻辑，而非抢播。  
> 代码开关：`CUSTOM_MUSIC_CAROUSEL = 1`（轮播）；`= 0`（抢播，当前未启用）。
>
> 产品行为：**A 正在播放 → B 点击播放 → 仍听 A → A 主动停止 → 切换到 B**  
> 分析结论：**与当前代码实现一致**。  
> 关联文档：[MULTIPOINT_一拖二逻辑.md](./MULTIPOINT_一拖二逻辑.md)、[BT_AUDIO_POLICY说明.md](./BT_AUDIO_POLICY说明.md)

---

## 1. 轮播 vs 抢播

| 模式 | 宏 | 行为 |
|------|-----|------|
| **轮播**（当前） | `CUSTOM_MUSIC_CAROUSEL = 1` | B 点播放 **不抢** A；A 停播后 **自然切到** B |
| 抢播（未启用） | `CUSTOM_MUSIC_CAROUSEL = 0` | B 点播放时应用层 **pause A + 切 primary 到 B**，立即出声 |

代码里 `_APP_AUDIO_PREEMPTION_`、`ui_audio_preemption_handler` 是历史命名（preemption），在轮播模式下仅做 **状态检测**，不执行 pause/切 primary。

---

## 2. 结论摘要

| 场景 | 产品行为 | 代码是否一致 | 主要实现层 |
|------|----------|--------------|------------|
| B 轮播等待（不抢 A） | B 播放无效，仍听 A | **一致** | BT 栈 Audio Policy + `CUSTOM_MUSIC_CAROUSEL=1` |
| A 停止后 B 出声 | A 停播后 B 自动出声 | **一致** | A2DP focus 释放 + AM resume |
| 双设备同时来电 | 按规则接听/拒接第二路 | **一致** | `app_econn_handle_call` + HFP |
| 通话中音乐 | 来电/通话可打断音乐 | **一致** | `audio_call_state` 策略表 |
| 双设备各有一路通话 | 仅一路 SCO，切换 active link | **一致** | HFP + `LM_change_active_link` |

---

## 3. 音乐轮播：与产品描述的对照

### 3.1 产品描述（轮播逻辑）

```
设备 A（Primary）正在播放
    ↓
设备 B 点击播放器「播放」
    ↓
耳机仍输出 A 的声音（B 排队等待，不抢播）
    ↓
设备 A 用户主动暂停/停止
    ↓
耳机切换到 B 的声音（轮到 B 播放）
```

### 3.2 代码验证：一致

实现分为 **BT 栈层（轮播的根本约束）** 和 **应用层（`CUSTOM_MUSIC_CAROUSEL` 确认轮播模式）** 两层。

---

## 4. BT 栈层：Audio Policy（轮播的核心）

文件：`wq-adk/components/bt_service/am/bt_audio_policy.c`

当 AM 处于 **MEDIA 状态**（设备 A 在播）时，设备 B 请求媒体焦点：

| B 的请求 opcode | 策略结果 | 含义 |
|-----------------|----------|------|
| `AUDIO_REQ_BT_MEDIA_ON` | `AUDIO_FOCUS_IGNORE` | B 不能启动前台媒体 |
| `AUDIO_REQ_BT_MEDIA_AVRCP_PLAYING` | `AUDIO_FOCUS_IGNORE` | B 点播放，不获得焦点（轮播等待） |
| `AUDIO_REQ_BT_MEDIA_LONG_DURATION` | `AUDIO_FOCUS_IGNORE` | 长时间播放请求也被忽略 |
| `AUDIO_REQ_BT_MEDIA_RESUME` | `AUDIO_FOCUS_GAIN` | A 释放焦点后，B resume 获焦点（轮到 B） |

```68:74:wq-adk/components/bt_service/am/bt_audio_policy.c
static const uint8_t audio_media_state[][AUDIO_POLICY_ST_COLS_NUM] = {
    {AUDIO_REQ_BT_MEDIA_ON,            AUDIO_FOCUS_IGNORE, AM_idle_ID       },
    {AUDIO_REQ_BT_MEDIA_AVRCP_PLAYING, AUDIO_FOCUS_IGNORE, AM_idle_ID       },
    {AUDIO_REQ_BT_MEDIA_LONG_DURATION, AUDIO_FOCUS_IGNORE, AM_idle_ID       },
    {AUDIO_REQ_BT_MEDIA_SILENT_DATA,   AUDIO_FOCUS_GAIN,   AM_media_ID      },
    {AUDIO_REQ_BT_MEDIA_RESUME,        AUDIO_FOCUS_GAIN,   AM_media_ID      },
```

**实际效果**：

- B 可能已 A2DP streaming，但 `focus != AUDIO_FOCUS_GAIN`，数据不进 `aud_sv_music_in()`，**无声**
- A 停止 → 释放 focus → B 走 `MEDIA_RESUME` → **轮到 B 出声**

---

## 5. 应用层：`CUSTOM_MUSIC_CAROUSEL` 轮播模式

### 5.1 宏定义（A2001 当前值）

| 宏 | 定义位置 | 当前值 | 作用 |
|----|----------|--------|------|
| `CUSTOM_MUSIC_CAROUSEL` | `app_econn_demo.c` | **`1`** | **轮播模式**：不主动 pause/切 primary |
| `_APP_AUDIO_PREEMPTION_` | `app_econn.h` | `1` | 双设备 streaming 状态检测框架（函数名含 preemption，轮播下仅检测） |

```185:187:wq-adk/project/a2001/acore/app/src/app_econn_demo.c
#ifndef CUSTOM_MUSIC_CAROUSEL
#define CUSTOM_MUSIC_CAROUSEL 1
#endif
```

### 5.2 轮播模式下应用层行为

`CUSTOM_MUSIC_CAROUSEL=1` 时，`ui_audio_preemption_handler()` 内 **pause / 切 primary 的代码被 `#ifndef` 裁掉**：

```8474:8502:wq-adk/project/a2001/acore/app/src/app_econn_demo.c
    if (双设备均在 A2DP STREAMING) {
        #ifndef CUSTOM_MUSIC_CAROUSEL   // =1 时不编译 ↓
            app_bt_pause(...);
            wq_bt_set_multipoint_primary(...);
        #endif
    }
```

仍保留的轮播相关逻辑：

| 机制 | 作用 |
|------|------|
| A2DP/AVRCP 状态检测 | 记录 `secondary_a2dp_in_start`、`primary_change_time` |
| `ECONN_MSG_ID_SLAVE_AVRCP_HANDLE_DELAY` | 双 PLAYING 时 **pause 副设备**，避免 UI 显示冲突 |
| `app_econn_player_play_silent_data_cb` | A 已 idle、B 在 streaming 时补偿 `wq_bt_set_multipoint_primary` |
| `anker_handle_sys_state` | A 从 streaming 降态时记录时间戳（轮播下不主动 `app_bt_play`） |

### 5.3 静默检测（轮播切换补偿）

`CUSTOM_MUSIC_CAROUSEL=1` 时注册静默检测回调：

```6333:6336:wq-adk/project/a2001/acore/app/src/app_econn_demo.c
    #ifdef CUSTOM_MUSIC_CAROUSEL
    aud_sv_silent_detect_notice_register(APP_SILENT_DETECT_TIME_MS,
                                         app_econn_player_play_silent_data_cb);
    #endif
```

- 条件：主设备 A2DP/AVRCP 仅 CONNECTED + 副设备 STREAMING
- 动作：切 primary 到副设备（A 已停、B 该轮播到的补偿）

---

## 6. 轮播逻辑流程图

```
┌─────────────────────────────────────────────────────────────────┐
│  A 正在播放 (Primary, AM=MEDIA, focus=GAIN)                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
              B 点击播放 (AVRCP PLAYING + A2DP start)
                             │
         ┌───────────────────┴───────────────────┐
         ▼                                       ▼
  BT Audio Policy                          应用层 (轮播)
  MEDIA + AVRCP_PLAYING                    CUSTOM_MUSIC_CAROUSEL=1
  → AUDIO_FOCUS_IGNORE                     → 检测状态，不 pause/不切 primary
         │                                       │
         ▼                                       ▼
  B 无声（轮播排队）              SLAVE_AVRCP_DELAY 可能 pause B 的 AVRCP
         │                                       │
         └───────────────────┬───────────────────┘
                             ▼
                    【用户仍听到 A】
                             │
              A 用户暂停/停止
                             │
                             ▼
              A 释放 focus → B: MEDIA_RESUME → GAIN
                             │
                             ▼
              可选：silent_detect_cb 切 primary 到 B
                             │
                             ▼
                    【听到 B — 轮到 B 轮播】
```

---

## 7. Primary 设备与 dev_list

- `dev_list[0]` = BT Primary（轮播期间通常保持为 **当前正在播的 A**）
- A 停、B resume 后，栈可能通过 `LM_change_active_link` 或 silent callback 将 primary 切到 B

---

## 8. 通话场景逻辑

（与轮播独立；通话优先级高于音乐。）

### 8.1 总体原则

| 规则 | 实现 |
|------|------|
| 通话优先级高于音乐 | `audio_media_state` 中来电 opcode → `AUDIO_FOCUS_GAIN` |
| 同时仅一路 SCO | HFP：新 SCO 时 disconnect 旧 SCO |
| Active Link 随通话切 | `LM_change_active_link()` |
| 双机通话按键 | `app_econn_handle_call()` + `app_ui_fun.c` |

### 8.2 音乐播放中来电

| 事件 | 策略结果 |
|------|----------|
| `AUDIO_REQ_BT_INBAND_RING` / `OUTBAND_RING` | `AUDIO_FOCUS_GAIN` → **打断音乐** |
| `AUDIO_REQ_BT_SPEECH_*` | `AUDIO_FOCUS_GAIN` → 通话建立 |

### 8.3 双设备通话要点

- **单 SCO**：新通话 disconnect 旧 SCO + 切 active link
- **双来电**：`first_incomingcall_phone_addr` 记录首路；双击接/三击拒 **第二路**
- **iPhone SCO**：`ECONN_MSG_ID_IPHONE_CONNECT_SCO` 补偿连接
- **通话中禁设一拖二**：返回 `APP_RESULT_IN_OUT_CALLING`

详细流程、代码引用、UI 按键表见原文档 §7 结构（内容不变，已从抢播文档迁移）。

---

## 9. 若改为抢播模式（对比参考）

将 `CUSTOM_MUSIC_CAROUSEL` 设为 `0` 后：

- `ui_audio_preemption_handler` 会 **pause 当前设备 + `wq_bt_set_multipoint_primary` 切到 B**
- B 点播放即可 **立即抢播**，不再等 A 停止
- 可能还需改 BT 策略表（影响面大）

**当前 A2001 保持 `CUSTOM_MUSIC_CAROUSEL=1`，即轮播逻辑。**

---

## 10. 关键源文件

| 文件 | 职责 |
|------|------|
| `bt_audio_policy.c` | 轮播：第二设备 MEDIA 请求 → IGNORE |
| `T_a2dp_top.c` | focus 申请/释放/数据路由 |
| `app_econn_demo.c` | **`CUSTOM_MUSIC_CAROUSEL`**、轮播检测、通话/SCO |
| `T_hfp_top.c` | 通话 focus、SCO 互斥 |
| `app_ui_fun.c` | 双设备通话按键 |

---

## 11. 验证用例（轮播）

| # | 步骤 | 预期 |
|---|------|------|
| 1 | A 播音乐 | 听到 A |
| 2 | B 点播放 | 仍听 A（轮播等待） |
| 3 | A 暂停 | 切到 B |
| 4 | A 播音乐，B 来电 | 铃声/通话打断 A |
| 5 | 双来电，双击耳机 | 接第二路 |
