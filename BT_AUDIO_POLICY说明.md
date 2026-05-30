# bt_audio_policy.c 说明

> 源文件：`wq-adk/components/bt_service/am/bt_audio_policy.c`  
> 头文件：`wq-adk/components/bt_service/am/bt_audio_policy.h`  
> 关联：[MULTIPOINT_轮播与通话逻辑.md](./MULTIPOINT_轮播与通话逻辑.md)

---

## 1. 文件是干什么的？

`bt_audio_policy.c` 是 BT 协议栈 **Audio Manager（AM）** 里的 **多连接音频焦点策略表**。

一句话：**当系统已经有一种音频在播（或通话中），新来的音频/通话请求能不能出声、出谁的声音，由这里的查表结果决定。**

它不负责连蓝牙、不负责解码 A2DP，只做 **决策**：

```
输入：当前 AM 状态（在播音乐？在通话？）+ 新请求的 opcode（谁、什么事件）
输出：AUDIO_FOCUS_*（给/不给焦点）+ 下一 AM 状态（dest state）
```

### 1.1 在调用链中的位置

```
A2DP/HFP/SPP 等模块
    设置 session.opcode = AUDIO_REQ_BT_xxx
    ↓
AM_request_audio_focus(session)
    ↓
__am_handle_policy_action()          [T_am_top.c]
    ↓
audio_policy_handle_action()         [bt_audio_policy.c]
    ├─ 1) audio_policy_handle_filter_action()  运行时 filter（可覆盖默认表）
    └─ 2) _audio_policy_handle_default_action()  默认策略表
    ↓
返回 focus = GAIN / IGNORE / GRANTING / LOSS
    ↓
若 GAIN → 启动 session，音频数据进 aud_sv
若 IGNORE → 请求被拒绝（一拖二轮播：第二台设备排队）
若 GRANTING → 需先 LM_change_active_link 切主链路
```

与 **应用层** `CUSTOM_MUSIC_CAROUSEL`（轮播）的关系：

- **Policy 层**：第二台设备 `AVRCP_PLAYING` 在 MEDIA 状态下 → `IGNORE`（栈层就不给焦点）
- **应用层**：`CUSTOM_MUSIC_CAROUSEL=1` 不主动 pause/切 primary（应用层轮播策略）

两层叠加，形成「A 在播时 B 无声，A 停后 B resume」的轮播行为。

---

## 2. 如何理解「很多宏/枚举」？

文件里看似宏很多，实际只有 **四类概念**，可以分开记：

| 类别 | 枚举前缀 | 含义 |
|------|----------|------|
| **行（当前状态）** | `AUDIO_POLICY_ST_*` | 查表时的「当前行」= AM 现在在什么场景 |
| **列（请求事件）** | `AUDIO_REQ_BT_*` | 查表时的「匹配键」= 谁发起了什么请求 |
| **结果（焦点）** | `AUDIO_FOCUS_*` | 查表输出：给不给音频焦点 |
| **下一状态** | `AM_*_ID` | 查表输出：AM session 切换到什么状态 |

每一张表的一行数据是 **三元组**：

```c
{ OPCODE,  AUDIO_FOCUS_RESULT,  DEST_STATE }
//  请求       焦点结果            下一 AM 状态
```

表结构列索引（`AUDIO_POLICY_ST_COLS_*`）：

```c
AUDIO_POLICY_ST_COLS_OPCODE      // 0：匹配键
AUDIO_POLICY_ST_COLS_RESULT      // 1：焦点结果
AUDIO_POLICY_ST_COLS_DEST_STATE  // 2：目标 AM 状态
```

---

## 3. 状态行（AUDIO_POLICY_ST_*）— 「当前在什么场景」

定义于 `bt_audio_policy.h` 的 `enum AUDIO_POLICY_ST_INDEX`，与 AM 状态基本一一对应：

| 索引枚举 | 对应 AM 状态 | 典型场景 |
|----------|--------------|----------|
| `AUDIO_POLICY_ST_IDLE` | `AM_idle_ID` | 无音频，空闲 |
| `AUDIO_POLICY_ST_BASE` | `AM_audio_base_ID` | 基础态（表为空） |
| `AUDIO_POLICY_ST_MEDIA` | `AM_media_ID` | **正在播放音乐（A2DP）** |
| `AUDIO_POLICY_ST_UP_STREAM` | `AM_up_stream_ID` | 上行（VAD/录音等） |
| `AUDIO_POLICY_ST_MEDIA_MIXED` | `AM_media_mixed_ID` | 音乐 + 上行混合 |
| `AUDIO_POLICY_ST_CALL` | `AM_call_ID` | 通话中（未 necessarily SCO speech） |
| `AUDIO_POLICY_ST_CALL_SPEECH` | `AM_call_speech_ID` | **SCO 语音通话进行中** |
| `AUDIO_POLICY_ST_CALL_INCOMMING` | — | 来电振铃中 |
| `AUDIO_POLICY_ST_CALL_OUTGOING` | — | 去电拨号中 |

HFP 子状态会由 `__am_get_audio_policy_state()` 进一步细分（来电/去电/ speech）。

**查表规则**：用 **当前活跃 session 的 ap_state** 作为 `audio_policy_st_table[]` 的下标，选中一张表（如 `audio_media_state`）。

---

## 4. 请求 opcode（AUDIO_REQ_BT_*）— 「发生了什么事件」

定义于 `enum AUDIO_POLICY_OPCODE`，由各 profile 在申请焦点前写入 `session.opcode`：

### 4.1 媒体（A2DP）

| Opcode | 典型触发 |
|--------|----------|
| `AUDIO_REQ_BT_MEDIA_ON` | A2DP 开始 streaming（手机 start） |
| `AUDIO_REQ_BT_MEDIA_AVRCP_PLAYING` | AVRCP 状态为 Playing |
| `AUDIO_REQ_BT_MEDIA_LONG_DURATION` | 长时间播放定时器到期 |
| `AUDIO_REQ_BT_MEDIA_RESUME` | 后台 session 恢复（**轮播：A 停后 B 常用此 opcode**） |
| `AUDIO_REQ_BT_MEDIA_SILENT_DATA` | 检测到静默 A2DP 数据 |
| `AUDIO_REQ_BT_MEDIA_ON_ONE` | 同设备第二 session（mixed 场景，`opcode++` 变体） |

### 4.2 通话（HFP）

| Opcode | 典型触发 |
|--------|----------|
| `AUDIO_REQ_BT_INBAND_RING` | 来电，带内振铃 |
| `AUDIO_REQ_BT_OUTBAND_RING` | 来电，带外振铃 |
| `AUDIO_REQ_BT_OUTGOING_CALL` | 去电 |
| `AUDIO_REQ_BT_CALL_RESUME` | 通话恢复 |
| `AUDIO_REQ_BT_SPEECH_INBAND_RING` | 通话中 inband ring |
| `AUDIO_REQ_BT_SPEECH_FROM_HF` | 耳机端发起语音（HF） |
| `AUDIO_REQ_BT_SPEECH_FROM_AG` | 手机端 AG 语音 |
| `AUDIO_REQ_BT_SPEECH_FROM_PC` | PC 语音 |
| `AUDIO_REQ_BT_SPEECH_VOICE_RECO` | 语音识别 |

### 4.3 上行 / 其它

| Opcode | 典型触发 |
|--------|----------|
| `AUDIO_REQ_BT_VAD_ON` / `VAD_ON_ONE` | 语音唤醒 VAD |
| `AUDIO_REQ_BT_RECORD_ON` / `RECORD_ON_ONE` | 录音 |

### 4.4 特殊 opcode

| 值 | 含义 |
|----|------|
| `AUDIO_REQ_DELEAGE` (0xFE) | **委托**：不直接给结果，跳转到另一张状态表继续查（见 §6） |
| `AUDIO_REQ_INVALID` (0xFF) | 表结束标记 |

---

## 5. 焦点结果（AUDIO_FOCUS_*）— 「查表输出什么意思」

```c
enum AUDIO_FOCUS_RESULT {
    AUDIO_FOCUS_NONE,      // 0：未命中 / 走默认；filter 未匹配时返回
    AUDIO_FOCUS_IGNORE,    // 1：拒绝，不能启动音频（轮播排队）
    AUDIO_FOCUS_LOSS,      // 2：失去焦点，应停止当前音频
    AUDIO_FOCUS_GRANTING,  // 3：准予但需先切 active link，暂不能播
    AUDIO_FOCUS_GAIN,      // 4：获得焦点，可以播/通话
};
```

| 结果 | 对 A2DP 的影响 | 对 HFP 的影响 |
|------|----------------|---------------|
| `GAIN` | `_a2dp_media_data` 送数据到 `aud_sv_music_in` | 可建立/继续 SCO |
| `IGNORE` | streaming 但 **无声** | 可能 reject SCO 或排队 |
| `GRANTING` | 等 `LM_change_active_link` 完成后再 GAIN | 同上 |
| `LOSS` | 停止送数据 | 关闭/暂停 session |

---

## 6. 查表算法

### 6.1 入口

```c
uint8_t audio_policy_handle_action(uint8_t state, uint8_t opcode, uint8_t *dest);
```

1. 先查 **运行时 filter**（`bt_audio_policy_filter.c`，可由 RPC/应用动态 `audio_policy_add_filter` 覆盖）
2. filter 返回 `AUDIO_FOCUS_NONE` 则查 **默认表** `_audio_policy_handle_default_action`

### 6.2 默认表遍历

```c
static const AUDIO_POLICY_ST_TBL audio_policy_st_table[] = {
    audio_idle_state,
    audio_base_state,
    audio_media_state,        // ← 一拖二轮播关键表
    audio_up_stream_state,
    audio_media_mixed_state,
    audio_call_state,
    audio_call_speech_state,
    audio_call_incoming_state,
    audio_call_outgoing_state,
};
```

逻辑（简化）：

```
table = audio_policy_st_table[state]
for each row in table:
    if row.opcode == opcode:
        *dest = row.dest_state
        return row.result
    if row.opcode == AUDIO_REQ_DELEAGE:
        state = row.dest_state   // 切换到另一张表
        table = audio_policy_st_table[state]
        重新从 index 0 查
return AUDIO_FOCUS_IGNORE   // 未匹配则拒绝
```

`AUDIO_REQ_DELEAGE` 用于 **来电/去电** 等子状态：当前表查不到完整规则时，委托到 `audio_call_state` 等基础表。

---

## 7. 各状态表要点（与一拖二/通话相关）

### 7.1 `audio_idle_state` — 空闲时

**几乎任何请求都给 GAIN**（媒体、通话、VAD 都能启动）。

### 7.2 `audio_media_state` — 正在播音乐（轮播核心）

| 新请求 | 结果 | 说明 |
|--------|------|------|
| `MEDIA_ON` / `AVRCP_PLAYING` / `LONG_DURATION` | **IGNORE** | **第二台设备不能抢播** |
| `MEDIA_RESUME` | **GAIN** | 当前设备恢复 / A 释放后 B resume |
| `INBAND_RING` / `OUTBAND_RING` | **GAIN** | **来电可打断音乐** |
| `SPEECH_*` | **GAIN** | 通话相关可打断 |
| `OUTGOING_CALL` / `CALL_RESUME` | IGNORE | 拨出/恢复通话在播音乐时不处理 |

→ 这就是一拖二 **轮播** 在栈层的根本依据。

### 7.3 `audio_call_state` — 通话中（未 speech）

| 新请求 | 结果 |
|--------|------|
| `MEDIA_ON` / `MEDIA_RESUME` | GAIN（通话结束可回音乐） |
| `INBAND_RING` | GAIN |
| `OUTBAND_RING` / `OUTGOING_CALL` | IGNORE |

### 7.4 `audio_call_speech_state` — SCO 语音中

Dreame 有定制注释（`wq fae dreame`）：

| 新请求 | 结果 | 备注 |
|--------|------|------|
| `MEDIA_ON` | IGNORE | 通话中不放音乐 |
| `OUTGOING_CALL` | **GAIN** | bug 1619 |
| `SPEECH_FROM_AG` | **GAIN** | bug 1723 |
| `INBAND_RING` / `SPEECH_INBAND_RING` | IGNORE | Dreame 定制 |

### 7.5 `audio_call_incoming_state` / `audio_call_outgoing_state`

来电/去电子状态表条目较少，复杂规则通过 `DELEAGE` 委托到 `audio_call_state`。

### 7.6 `audio_media_mixed_state`

音乐 + VAD/录音混合时，`MEDIA_ON_ONE` → IGNORE；`DELEAGE` → 回到 up_stream。

---

## 8. 运行时 Filter（可覆盖默认表）

文件：`bt_audio_policy_filter.c`

```c
// 示例：media 状态下的 filter（可被 audio_policy_add_filter 挂到 AUDIO_POLICY_ST_MEDIA）
media_state_filter[] = {
    { AVRCP_PLAYING,  GAIN,  AM_idle_ID },  // 若启用 filter，可改变默认 IGNORE
    { LONG_DURATION,  GAIN,  AM_idle_ID },
};

call_speech_state_filter[] = {
    { INBAND_RING,    IGNORE, AM_call_ID },  // speech 中忽略部分来电
    ...
};
```

`audio_policy_handle_action` **优先** 使用 filter；只有 filter 返回 `AUDIO_FOCUS_NONE` 才用 `bt_audio_policy.c` 默认表。

应用/DTOP 可通过 RPC `bt_cmd_audio_policy_filter_t` → `audio_policy_add_filter()` 动态改策略（见 `app_user_cmd.c`）。

---

## 9. 与 bt_audio_user_action.c 的区别

| 文件 | 阶段 | 输入 | 作用 |
|------|------|------|------|
| **bt_audio_policy.c** | 申请焦点 **之前** | `AUDIO_REQ_BT_*` | 能不能获得 focus |
| **bt_audio_user_action.c** | 用户/系统 **操作** | `AM_USER_*` | 暂停/挂断后 resume 谁、释放哪个 session |

Policy 回答「能不能播」；User action 回答「停播之后怎么办」。

---

## 10. 实战：读一条表项

**场景**：一拖二，A 正在播音乐，B 点击播放。

1. 当前活跃 session → `ap_state = AUDIO_POLICY_ST_MEDIA`
2. B 的 A2DP 设 `opcode = AUDIO_REQ_BT_MEDIA_AVRCP_PLAYING`
3. 查 `audio_media_state[]` 对应行：

```c
{AUDIO_REQ_BT_MEDIA_AVRCP_PLAYING, AUDIO_FOCUS_IGNORE, AM_idle_ID},
```

4. 返回 `IGNORE` → B **不能** 获得焦点 → A2DP 数据不进音频通路 → **仍听 A**

**A 暂停后**：

1. A release session → AM → idle
2. B 后台 session：`opcode = AUDIO_REQ_BT_MEDIA_RESUME`
3. 查 `audio_idle_state` 或 media 下 RESUME 行 → `GAIN` → **B 出声**

---

## 11. 宏/枚举速查总表

```
┌─────────────────────────────────────────────────────────────┐
│  audio_policy_handle_action(state, opcode, &dest)           │
├─────────────────────────────────────────────────────────────┤
│  state  ← AUDIO_POLICY_ST_*  (当前 AM/活跃 session 场景)     │
│  opcode ← AUDIO_REQ_BT_*       (A2DP/HFP 发起的事件)         │
├─────────────────────────────────────────────────────────────┤
│  返回值 ← AUDIO_FOCUS_*        (GAIN/IGNORE/GRANTING/LOSS)   │
│  dest   ← AM_*_ID              (session 下一状态)            │
└─────────────────────────────────────────────────────────────┘

表 = audio_policy_st_table[state][行]
行 = { opcode, result, dest_state }
```

---

## 12. 相关源文件

| 文件 | 职责 |
|------|------|
| `bt_audio_policy.c` | 默认策略表 + 查表逻辑 |
| `bt_audio_policy.h` | 枚举、API 声明 |
| `bt_audio_policy_filter.c` | 运行时 filter + add/remove API |
| `T_am_top.c` | `__am_handle_policy_action`、`AM_request_audio_focus` |
| `T_a2dp_top.c` | 设置 media opcode、根据 focus 送数据 |
| `T_hfp_top.c` | 设置 call opcode、SCO 与 focus |
| `bt_audio_user_action.c` | 用户操作后的 session 状态机 |

---

## 13. 修改策略时的注意点

1. **改轮播/抢播**：重点看 `audio_media_state` 里 `AVRCP_PLAYING` / `MEDIA_ON` 是 `IGNORE` 还是 `GAIN`
2. **改通话打断音乐**：看 `audio_media_state` 里 `INBAND_RING` 等是否为 `GAIN`
3. **Dreame 定制**：`audio_call_speech_state` 中带 `wq fae dreame` 注释的行
4. **动态覆盖**：优先走 filter，改默认表可能被 `audio_policy_add_filter` 覆盖
5. **与应用层一致**：栈层改成 GAIN 后，应用层 `CUSTOM_MUSIC_CAROUSEL` 仍可能 pause/不 pause，需两层一起看
