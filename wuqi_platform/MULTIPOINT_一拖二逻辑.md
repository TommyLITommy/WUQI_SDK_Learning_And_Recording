# A2001 一拖二（双设备连接）逻辑梳理

> 一拖二：一副 TWS 耳机同时连接 **两台手机**（Bluetooth Multipoint），可在两台设备间切换音频/通话主设备。
>
> 代码主路径：`wq-adk/project/a2001/acore/app/src/`

---

## 1. 核心概念

| 概念 | 变量/API | 说明 |
|------|----------|------|
| 一拖二开关 | `usr_cfg_get/set_multi_conn_enabled()` | 运行时开关 |
| 持久化 | `mapp_setup.gmulti_conn_enable` | Flash 存储，key: `cmulti_conn_enable` |
| 最大连接数 | `app_conn_set/get_bt_max_connection()` | 关闭=1，开启=2 |
| 主设备 (Primary) | BT 栈内部 active link | 决定当前音频/通话路由 |
| 副设备 (Secondary) | 另一台已连接手机 | 保持连接但不活跃播放 |
| PDL | `usr_cfg_pdl_*()` | 配对设备列表，最多 8 条历史 |

### 1.1 与 LDAC 互斥

- 开启一拖二 → 强制关闭 LDAC（`usr_cfg_set_codec_ldac_enabled(0)`）
- LDAC 开启时，状态上报中一拖二字段固定为 `0`
- 若开启一拖二时 LDAC 已开，会触发 `EVTUSR_REBOOT` 重启

### 1.2 开机初始化

`mapp_setup_ini()`（`app_cmd.c`）读取 `gmulti_conn_enable`，同步到：

```c
if (mapp_setup.gmulti_conn_enable) {
    usr_cfg_set_multi_conn_enabled(1);
    app_conn_set_bt_max_connection(2);
} else {
    usr_cfg_set_multi_conn_enabled(0);
    app_conn_set_bt_max_connection(1);
}
```

默认出厂配置：`gmulti_conn_enable = 1`（一拖二默认开启）。

---

## 2. 协议 API

项目存在 **两套协议入口**，业务逻辑最终汇聚到同一组处理函数。

### 2.1 SoundTool 协议（Group `0xAB`）

文档参考：`SoundTool_Protocol.md` §3.8  
处理入口：`app_econn_demo.c` → `handle_pdl_set_cmd()` / `handle_pdl_get_cmd()`

| CMD ID | 名称 | 方向 | 数据 | 处理函数 |
|--------|------|------|------|----------|
| `0x51` | 获取连接历史 | APP→Device | 无 | `handle_pdl_get_cmd(PDL_GET_DATA)` |
| `0xF1` | 断开指定设备 | APP→Device | MAC 6B | `handle_pdl_disconn_specifical_dev_cmd()` |
| `0xF2` | 连接指定设备 | APP→Device | MAC 6B | `handle_pdl_conn_specifical_dev_cmd()` |
| `0xF3` | 删除历史设备 | APP→Device | MAC 6B | `handle_pdl_del_specifical_dev_cmd()` |
| **`0xF4`** | **设置一拖二开关** | APP→Device | `0x00`关 / `0x01`开 | `handle_pdl_set_mode_cmd()` → **重启** |
| **`0xF5`** | **连接新设备** | APP→Device | 无 | `ECONN_MSG_ID_2RD_DEVICE_PAIRING` |
| `0xF6` | 上报手机名称 | APP→Device | 名称 32B | 更新 `econn_cfg.gble_phonename` |

**状态上报**（设备信息扩展字段 `param[78]`，`app_econn_demo.c`）：

```c
param[78] = (usr_cfg_get_codec_ldac_enabled() == true) ? 0 : usr_cfg_get_multi_conn_enabled();
```

SoundTool 全量状态包中，扩展模块第 5 字节为「一拖二状态」。

### 2.2 自定义协议（Group `0x02` / `cgroup_more_comm`）

定义：`app_protocol.h`  
处理：`app_cmd.c`、`app_cmd_from_msg.c`（TWS 从耳同步）

| Sub ID | 宏名 | 数据 | 说明 |
|--------|------|------|------|
| **`0x11`** | `cgroup0x02_sub_id_switch` | `0/1` | 设置一拖二开关 |
| **`0x12`** | `cgroup0x02_sub_id_enter_switch` | 无 | 进入第二设备配对 |
| `0x13` | `cgroup0x02_sub_id_phonename` | 字符串 | 设置手机名称 |
| `0x14` | `cgroup0x02_sub_id_keylist` | — | 设备列表（`mdevice_keylist_rsp`） |
| `0x15` | `cgroup0x02_sub_id_disconnect_by_mac` | MAC 6B | 按 MAC 断开 |
| `0x16` | `cgroup0x02_sub_id_connect_by_mac` | MAC 6B | 按 MAC 连接 |
| `0x17` | `cgroup0x02_sub_id_delete_by_mac` | MAC 6B | 按 MAC 删除 PDL |
| `0x18` | `cgroup0x02_sub_id_device_media_status` | — | 主设备媒体状态上报 |

### 2.3 AI 辅助协议（Group `0xA1`）

| Sub ID | 宏名 | 说明 |
|--------|------|------|
| `0x91` | `cgroup0xa1_ai_assist_set_force` | 强制切换 multipoint 主设备（见 §4） |

---

## 3. 业务逻辑流程

### 3.1 打开/关闭一拖二

**入口**：`cgroup0x02_sub_id_switch`（0x11）或 SoundTool `0xF4`

**前置检查**：
- 通话中（`app_bt_get_sys_state(NULL) >= STATE_INCOMING_CALL`）→ 返回 `APP_RESULT_IN_OUT_CALLING`

**开启（data = 1）**：

```
usr_cfg_set_multi_conn_enabled(1)
app_conn_set_bt_max_connection(2)
app_bt_set_discoverable_and_connectable(false, true)
usr_cfg_set_codec_ldac_enabled(0)
reconnect_last_devices()          // 尝试重连上次被断开的第二台设备
mapp_setup_storage(cmulti_conn_enable, &1)
```

**关闭（data = 0）**：

```
usr_cfg_set_multi_conn_enabled(0)
app_conn_set_bt_max_connection(1)
if (app_bt_get_connected_count() > 1)
    disconnect_unconnected_devices()   // 断开多余连接，记录 last_multi_addr
ECONN_MSG_ID_2RD_PAIRING_TIMEOUT       // 500ms 后关闭 discoverable
mapp_setup_storage(cmulti_conn_enable, &0)
mdevice_keylist_rsp(1)                 // 上报设备列表
```

**SoundTool 0xF4 差异**：`handle_pdl_set_mode_cmd()` 成功后 `app_evt_send_delay(EVTUSR_REBOOT, 1000)` 重启；TWS 主耳通过 `ECONN_MSG_ID_PDL_ON_OFF` 同步从耳。

### 3.2 连接第二台设备

**入口**：`0xF5` / `cgroup0x02_sub_id_enter_switch`（0x12）

**消息**：`ECONN_MSG_ID_2RD_DEVICE_PAIRING`

**条件**（全部满足才进入配对）：
- `app_bt_get_connected_count() == CONFIG_APP_BT_MAX_CONNECTION - 1`（已连 1 台）
- `!app_bt_is_discoverable()`
- `usr_cfg_get_multi_conn_enabled() == true`

**动作**：
- 发送 `ECONN_MSG_ID_FORCE_DISCOVERABLE` 进入可发现模式
- TWS 同步从耳
- 回复 APP：`PDL_CON_NEW_DEV_CMD`，data `0x01`=成功 / `0x00`=失败

**配对超时**：`ECONN_MSG_ID_2RD_PAIRING_TIMEOUT` → 关闭 discoverable，重置 `force_discoverable`。

### 3.3 按 MAC 连接/断开/删除

| 操作 | 函数 | 底层 API | 异步上报 |
|------|------|----------|----------|
| 连接 | `handle_pdl_conn_specifical_dev_cmd()` | `app_bt_connect(addr)` | 12s 后 `ECONN_MSG_ID_REPORT_CONN_RESULT` |
| 断开 | `handle_pdl_disconn_specifical_dev_cmd()` | `app_bt_disconnect(addr)` | 5s 后 `ECONN_MSG_ID_REPORT_DISCONN_RESULT` |
| 删除 | `handle_pdl_del_specifical_dev_cmd()` | `usr_cfg_pdl_remove(addr)` | — |
| 重连历史 | `reconnect_last_devices()` | 上述 connect + 8s/20s 重试消息 | — |

MAC 连接额外触发：
- `ECONN_MSG_ID_RETRY_CONNECT_BY_MAC`（8s 重试）
- `ECONN_MSG_ID_RECONNECT_FAILED_REPORT`（20s 失败上报）

### 3.4 关闭一拖二时清理连接

`disconnect_unconnected_devices()`（`app_cmd.c`）：
- 遍历 PDL 中所有设备
- 断开「未连接 / 非当前 SPP 连接 / 名称不匹配」的设备
- 记录 `last_multi_addr` 供后续 `reconnect_last_devices()` 使用

### 3.5 用户按键触发

| 事件 | 条件 | 行为 |
|------|------|------|
| `EVTUSR_ENTER_PAIRING` | 一拖二开 + 已连 1 台 | `ECONN_MSG_ID_2RD_DEVICE_PAIRING` |
| `EVTUSR_ENTER_PAIRING` | 其他 | `app_bt_disconnect_all()` 后普通配对 |
| `EVTUSR_CUSTOM_8` + 双耳长按 | 一拖二开 + 已连 1 台 + 非充电盒/非 discoverable | `ECONN_MSG_ID_2RD_DEVICE_PAIRING` |

### 3.6 TWS 主从同步

| 消息 | 作用 |
|------|------|
| `ECONN_MSG_ID_PDL_ON_OFF` | 主耳同步一拖二开关到从耳，从耳重启 |
| `app_cmd_from_msg.c` | 从耳处理 Group 0x02 各 sub_id |

### 3.7 主设备媒体状态上报

`report_mulit_device_media_status()`（`app_ui_fun.c`）通过 `0x02/0x18` 上报：

```c
typedef struct {
    uint8_t prev_primary_status;   // multi_status_t: IDLE/PLAY/PAUSE/PHONECALL
    BD_ADDR_T prev_primary_addr;   // 当前主设备 MAC
} primary_device_info_t;
```

---

## 4. `wq_bt_set_multipoint_primary` API 详解

### 4.1 函数原型

```c
// wq-adk/components/bt_rpc/inc/acore/wq_bt.h
/**
 * @brief set primary device of multipoint (extension)
 * @param addr  目标主设备的蓝牙 MAC 地址
 * @param force 是否强制切换主设备
 * @return WQ_RET_OK 成功；WQ_RET_NOT_EXIST 地址无效
 */
WQ_RET wq_bt_set_multipoint_primary(const BD_ADDR_T *addr, bool force);
```

### 4.2 调用链

```
应用层 wq_bt_set_multipoint_primary(addr, force)
    ↓ wq_send_rpc_cmd(BT_CMD_SET_MULTIPOINT_PRIMARY, ...)
BT Service bt_service_set_multipoint_primary()
    ↓ FOUND_VALID_ID_BY_ADDR → CM 连接管理索引
    ↓ LM_change_active_link(index, USER_TASK_ID())
Link Manager LM_change_active_link()
    ↓ RT_MSG: LM_REQUEST_CHANGE_ACTIVE_LINK_SIG
BT 栈切换 active link（主链路）
    ↓ 切换成功
bt_service_evt_multipoint_primary_changed(bd_addr, result)
    ↓ RPC 事件 BT_EVT_MULTIPOINT_PRIMARY_CHANGED
app_bt.c: bt_evt_multipoint_primary_changed_handler()
    ↓ app_econn_handle_multipoint_primary_changed()
    ↓ app_audio_handle_primary_changed()
    ↓ usr_cfg_pdl_add_first(&addr)  // 主设备置顶 PDL
```

### 4.3 RPC 数据结构

```c
// bt_rpc_api.h
typedef struct {
    IN BD_ADDR_T addr;
    IN bool force;
} bt_cmd_set_multipoint_primary_t;

typedef struct {
    BD_ADDR_T addr;
    uint8_t result;   // bt_primary_changed_result_t
} bt_evt_multipoint_primary_changed_t;
```

### 4.4 返回值与事件结果

**RPC 命令返回**（`bt_service_set_multipoint_primary`）：

| 返回值 | 含义 |
|--------|------|
| `BT_RESULT_SUCCESS` | 已发起主链路切换 |
| `BT_RESULT_ALREADY_EXISTS` | 该设备已是 primary |
| `BT_RESULT_NOT_EXISTS` | MAC 不在已连接设备列表中 |

**切换结果事件**（`BT_EVT_MULTIPOINT_PRIMARY_CHANGED`）：

| result 枚举 | 含义 |
|-------------|------|
| `PRIMARY_CHANGED_RESULT_SUCCESS` (0) | 切换成功 |
| `PRIMARY_CHANGED_RESULT_TWS_TDS` (1) | TWS 角色切换进行中，暂不可切 |
| `PRIMARY_CHANGED_RESULT_ACTIVITY` (2) | 当前有活动业务阻止切换 |
| `PRIMARY_CHANGED_RESULT_OTHER_ERROR` (0xFF) | 其他错误 |

### 4.5 `force` 参数含义

- `force = true`：强制将指定设备设为主设备，即使当前有其他设备在播放
- `force = false`：非强制切换（如 GA 连接时自动设为主设备）

BT Service 实现中 **未直接使用 `force` 字段**，实际强制行为由上层在调用前 pause 另一台设备来保证。

### 4.6 A2001 项目中的调用场景

| 位置 | 场景 | 参数 |
|------|------|------|
| `app_cmd.c` `0xA1/0x91` | APP 强制切换主设备（双连且 ≥2 台） | IAP2 连接时 → IAP2 地址；否则 → `remote_addr`；`force=true` |
| `app_econn_demo.c` `app_econn_player_play_silent_data_cb` | 主设备 A2DP+AVRCP 已连、副设备 A2DP streaming | 切到副设备 `slave_addr`，`force=true` |
| `app_econn_demo.c` `ui_audio_preemption_handler` | 双设备同时 A2DP streaming（`CUSTOM_MUSIC_CAROUSEL=0` 时） | 抢播：pause + 切 primary；**当前 `=1` 轮播模式不执行**，见 [MULTIPOINT_轮播与通话逻辑.md](./MULTIPOINT_轮播与通话逻辑.md) |
| `app_bt.c` GA 连接回调 | 新 GA 设备连接且非 current | 设为新 primary，`force=false` |

### 4.7 事件回调处理

**BT 层**（`app_bt.c`）：

```c
static void bt_evt_multipoint_primary_changed_handler(...)
{
    if (param->result == PRIMARY_CHANGED_RESULT_SUCCESS) {
        current = device;                    // 更新当前主设备指针
        generate_sys_state();
        usr_cfg_pdl_add_first(&param->addr); // 主设备移到 PDL 首位
        app_audio_handle_primary_changed();  // 音频路由切换
    }
    app_econn_handle_multipoint_primary_changed(&param->addr, param->result);
}
```

**应用层**（A2001 `app_econn_demo.c`）：

```c
void app_econn_handle_multipoint_primary_changed(const BD_ADDR_T *addr, uint8_t result)
{
    // A2001 中为空实现，可在此扩展 UI/上报逻辑
}
```

### 4.8 辅助函数：获取双设备地址

```c
// app_econn_demo.c
static bool app_econn_get_multi_dev_addr(BD_ADDR_T *primary_addr, BD_ADDR_T *secondary_addr)
{
    BD_ADDR_T dev_list[APP_BT_MAX_CONNECTION];
    if (app_bt_get_connected_list(dev_list, APP_BT_MAX_CONNECTION) > 1) {
        memcpy(primary_addr,   &dev_list[0], sizeof(BD_ADDR_T));  // primary
        memcpy(secondary_addr, &dev_list[1], sizeof(BD_ADDR_T));  // secondary
        return true;
    }
    return false;
}
```

`dev_list[0]` 为 BT 栈认定的 primary，`dev_list[1]` 为 secondary。

---

## 5. 内部消息 ID

定义：`app_econn_demo.h`

| 消息 ID | 值 | 作用 |
|---------|-----|------|
| `ECONN_MSG_ID_2RD_DEVICE_PAIRING` | 26 | 第二设备配对 |
| `ECONN_MSG_ID_2RD_PAIRING_TIMEOUT` | 27 | 配对超时收尾 |
| `ECONN_MSG_ID_PDL_ON_OFF` | — | TWS 同步一拖二开关 |
| `ECONN_MSG_ID_REPORT_CONN_RESULT` | — | 连接结果上报 APP |
| `ECONN_MSG_ID_REPORT_DISCONN_RESULT` | — | 断开结果上报 APP |
| `ECONN_MSG_ID_RETRY_CONNECT_BY_MAC` | — | MAC 连接重试（8s） |
| `ECONN_MSG_ID_RECONNECT_FAILED_REPORT` | — | 重连失败上报（20s） |
| `ECONN_MSG_ID_FORCE_DISCOVERABLE` | — | 强制进入可发现模式 |

---

## 6. 底层 API 汇总

### 6.1 配置层

```c
// usr_cfg.h
bool usr_cfg_get_multi_conn_enabled(void);
void usr_cfg_set_multi_conn_enabled(uint8_t enabled);
void usr_cfg_pdl_add_first/add/remove/get_first/get_next/get_list(...);

// app_conn.h
void app_conn_set_bt_max_connection(uint8_t max_connection);
uint8_t app_conn_get_bt_max_connection(void);
```

### 6.2 BT 连接层

```c
// app_bt.h
WQ_RET app_bt_connect(const BD_ADDR_T *addr);
WQ_RET app_bt_disconnect(const BD_ADDR_T *addr);
WQ_RET app_bt_disconnect_all(void);
WQ_RET app_bt_pause(const BD_ADDR_T *addr);
uint8_t app_bt_get_connected_count(void);
uint8_t app_bt_get_connected_list(BD_ADDR_T *addr_list, uint8_t max_count);
bool app_bt_is_device_connected(const BD_ADDR_T *addr);
WQ_RET app_bt_set_discoverable(bool discoverable);
WQ_RET app_bt_set_discoverable_and_connectable(bool discoverable, bool connectable);
WQ_RET app_bt_enter_ag_pairing(void);
uint32_t app_bt_get_sys_state(const BD_ADDR_T *addr);
bt_a2dp_state_t app_bt_get_a2dp_state(const BD_ADDR_T *addr);
bt_avrcp_state_t app_bt_get_avrcp_state(const BD_ADDR_T *addr);
```

### 6.3 Multipoint 层

```c
// wq_bt.h
WQ_RET wq_bt_set_multipoint_primary(const BD_ADDR_T *addr, bool force);
WQ_RET wq_bt_a2dp_notify_data_silent(void);
```

### 6.4 应用封装层

```c
// app_cmd.h / app_cmd.c
bool handle_pdl_set_mode_cmd(const uint8_t *param, uint8_t len);
bool handle_pdl_conn_specifical_dev_cmd(const uint8_t *param, uint8_t len);
bool handle_pdl_disconn_specifical_dev_cmd(const uint8_t *param, uint8_t len);
bool handle_pdl_del_specifical_dev_cmd(const uint8_t *param, uint8_t len);
uint8_t reconnect_last_devices(void);
uint8_t disconnect_unconnected_devices(void);
void mdevice_keylist_rsp(uint8_t Role);

// app_ui_fun.h
void report_mulit_device_media_status(uint8_t is_active_report);

// app_econn_demo.h
bool get_multi_dev_addr(BD_ADDR_T *primary_addr, BD_ADDR_T *secondary_addr);
void app_econn_handle_multipoint_primary_changed(const BD_ADDR_T *addr, uint8_t result);
```

---

## 7. 整体流程图

```
┌─────────────────────────────────────────────────────────────┐
│                        APP / 按键 / 充电盒                    │
└──────────────┬──────────────────────────┬───────────────────┘
               │                          │
    SoundTool 0xAB                   自定义 0x02
    (F4/F5/F1~F3)                   (0x11~0x18)
               │                          │
               └──────────┬───────────────┘
                          ▼
              handle_pdl_* / app_cmd switch
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
   usr_cfg multi_conn  app_conn max=2   mapp_setup 持久化
          │               │
          ▼               ▼
   第二设备配对      按 MAC 连接/断开
   (2RD_DEVICE_      (app_bt_connect/
    PAIRING)           disconnect)
          │               │
          └───────┬───────┘
                  ▼
         BT 栈双设备连接 (CM/LM)
                  │
                  ▼
    wq_bt_set_multipoint_primary()  ← 音频抢占 / APP 强制切换
                  │
                  ▼
    BT_EVT_MULTIPOINT_PRIMARY_CHANGED
                  │
          ┌───────┴───────┐
          ▼               ▼
   app_audio_*      report 0x02/0x18
   音频路由切换      主设备状态上报 APP
```

---

## 8. 关键源文件索引

| 文件 | 职责 |
|------|------|
| `app_cmd.c` | 自定义协议 Group 0x02 一拖二全流程 |
| `app_cmd_from_msg.c` | TWS 从耳协议同步 |
| `app_econn_demo.c` | SoundTool 0xAB、配对/超时/按键/音频抢占 |
| `app_ui_fun.c` | 主设备媒体状态检测与 0x18 上报 |
| `app_protocol.h` | Group 0x02 sub_id 定义 |
| `SoundTool_Protocol.md` | SoundTool 0xAB 协议文档 |
| `wq_bt.c` / `wq_bt.h` | `wq_bt_set_multipoint_primary` RPC 封装 |
| `app_user_cmd.c` | BT Service：`bt_service_set_multipoint_primary` |
| `lm_top.c` | `LM_change_active_link` 主链路切换 |
| `app_bt.c` | `BT_EVT_MULTIPOINT_PRIMARY_CHANGED` 事件处理 |
| `usr_cfg.h` | multi_conn / PDL 配置 API |
| `app_conn.h` | bt_max_connection 管理 |
| `bt_rpc_api.h` | RPC 命令/事件数据结构定义 |

---

## 9. 注意事项

1. **通话中禁止**开关一拖二（两种协议均检查 `STATE_INCOMING_CALL`）。
2. **LDAC 与一拖二互斥**，切换时需重启固件。
3. SoundTool `0xF4` 与自定义 `0x11` 行为略有差异：`0xF4` 必定重启，`0x11` 仅在 LDAC 冲突时重启。
4. `wq_bt_set_multipoint_primary` 的 `force` 参数在 BT Service 层未单独处理，强制切换依赖上层先 `app_bt_pause()` 暂停竞争设备。
5. `app_econn_handle_multipoint_primary_changed()` 在 A2001 为空实现，主设备变更的 APP 通知主要通过 `0x18` 媒体状态上报和 PDL 列表刷新实现。
