# 物奇 TWS 量产烧录对耳 MAC 后自动组队流程梳理

本文记录 A2007 工程中“物奇烧录上位机指定对耳 MAC，两个耳机启动后自动组队”对应的代码链路，方便后续学习和排查。

## 1. 结论先行

量产烧录时，上位机会把对耳 MAC 等信息写入 `ro_cfg.xml` 对应的 `bt_general_data` 配置。设备启动后，BT service 先把这份配置加载到 `bt_srv_general_data`，随后在蓝牙 enable 时根据 `peer_addr` 和 `tws_role_mode/is_pri_dev` 决定本机是 PRI 还是 SEC，并设置 `PRI_BDADDR()/SEC_BDADDR()`。RM 模块看到有效 `PEER_BDADDR()` 后自动发起 `tws_link_connect()`，TWS 连接成功后写回 `peer_addr`，并把 `tws_single_mode` 切到双耳模式。

核心链路：

```text
烧录上位机
  -> ro_cfg.xml / bt_general_data.peer_addr
  -> bt_storage_load()
  -> bt_storage_get_peer_addr()
  -> bt_service_power_on_off()
  -> 设置 HEADSET_PRI / HEADSET_SEC 与 PRI_BDADDR / SEC_BDADDR
  -> RM 模块启动 TWS connect
  -> tws_link_connect(PEER_BDADDR())
  -> 连接成功后保存 peer_addr + STORAGE_TWS_DOUBLE_MODE
```

## 2. 量产烧录写入的数据

工程里的配置模板位于：

- `wq-adk/project/a2007/config/7036A/ro_cfg.xml`
- `wq-adk/project/a2007/config/7036AC/ro_cfg.xml`
- `wq-adk/project/a2007/config/7035AX-B/ro_cfg.xml`

典型字段如下：

```xml
<ro_cfg_bt_general_data_t name="bt_general_data">
  <tws_linkkey type="bytes" len="$R0_BT_LINK_KEY_SIZE" input="hex">00000000000000000000000000000000</tws_linkkey>
  <is_pri_dev type="uint8" input="check">0</is_pri_dev>
  <peer_addr type="bytes" len="6" input="hex">000000000000</peer_addr>
  <tws_single_mode type="uint8">0</tws_single_mode>
</ro_cfg_bt_general_data_t>

<ro_cfg_bt_readonly_data_t name="bt_readonly_data">
  <value type="bits" len="8">
    <is_left_dev type="uint8" len="1">1</is_left_dev>
    <tws_role_mode type="uint8" len="2">1</tws_role_mode>
  </value>
</ro_cfg_bt_readonly_data_t>
```

字段含义：

| 字段 | 含义 |
| --- | --- |
| `peer_addr` | 对耳 MAC。左耳烧右耳 MAC，右耳烧左耳 MAC。 |
| `tws_single_mode` | `0` 表示双耳模式，`1` 表示单耳模式。 |
| `tws_linkkey` | TWS 之间的 link key，可预烧录；不预烧录时首次连接后可生成并保存。 |
| `is_pri_dev` | 可写区的主从标志，`1` 为 PRI，`0` 为 SEC。 |
| `tws_role_mode` | 只读区主从模式，`0` 自动/使用 general info，`1` 强制 PRI，`2` 强制 SEC。优先级高于 `is_pri_dev`。 |
| `is_left_dev` | 左右耳标记，影响左右声道等逻辑。 |

注意：`peer_addr = 00:00:00:00:00:00` 表示未指定对耳；`peer_addr = FF:FF:FF:FF:FF:FF` 在 RM 中被定义为 single/factory mode 对耳地址。

## 3. 数据结构对应关系

BT service 侧结构体在：

`wq-adk/components/bt_service/common/bt_srv_storage.h`

```c
typedef struct _bt_general_data {
    uint8_t tws_link_key[BT_LINK_KEY_SIZE];
    uint8_t is_pri_dev;
    uint8_t peer_addr[BT_BD_ADDR_SIZE];
    uint8_t tws_single_mode;
    uint8_t resrved1;
    uint16_t resrved2;
    uint32_t resrved3;
    uint32_t resrved4;
} bt_general_data_t;
```

只读区结构体：

```c
typedef struct _bt_readonly_data {
    uint8_t is_left_dev : 1,
        tws_role_mode : 2, resrved_5bits : 5;
    ...
} bt_readonly_data_t;
```

A-core 应用层也有一份同布局结构，在：

`wq-adk/components/apps/acore/bt/src/app_bt.c`

用于从 `BT_GENERAL_KV_ID` 读取 `peer_addr`，避免把对耳地址当作普通手机配对设备写入 PDL。

## 4. BT service 启动时加载烧录数据

BT service 初始化入口在：

`wq-adk/components/bt_service/common/T_appl_main.c`

关键调用：

```c
bt_storage_load();
```

真正加载在：

`wq-adk/components/bt_service/common/bt_srv_storage.c`

```c
static void _bt_general_data_load(void)
{
    uint16_t target_data_len = sizeof(bt_general_data_t);
    ret = bt_service_storage_get(BT_SRV_STORAGE_GENERAL_DATA_ID,
                                 (uint8_t *)&bt_srv_general_data,
                                 &target_data_len);
    if (ret) {
        memcpy_s(&bt_srv_general_data, sizeof(bt_general_data_t),
                 &(ro_cfg()->bt_general_data),
                 sizeof(ro_cfg()->bt_general_data));
    }
}
```

逻辑说明：

1. 优先从可写 storage 读 `BT_SRV_STORAGE_GENERAL_DATA_ID`。
2. 如果 storage 里没有有效数据，则回退读取 `ro_cfg()->bt_general_data`。
3. 只读配置 `bt_readonly_data` 直接从 `ro_cfg()->bt_readonly_data` 拷贝。
4. 加载后日志会打印 `tws_link_key/is_left/tws_role_mode/is_pri/tws_single_mode/peer_addr`。

## 5. 主从角色和地址映射

蓝牙 enable 的 RPC 处理在：

`wq-adk/components/bt_service/bt_rpc/app_user_cmd.c`

函数：`bt_service_power_on_off()`

关键逻辑：

```c
const uint8_t *peer_addr = bt_storage_get_peer_addr();
bool force_pri = !RM_IS_VALID_PEER_ADDR(peer_addr);

if (bt_storage_is_pri_dev() || force_pri) {
    SET_LOCAL_ROLE(HEADSET_PRI);
    BT_CPY_BD_ADDR(PRI_BDADDR(), info->local_addr.addr);
    BT_CPY_BD_ADDR(SEC_BDADDR(), peer_addr);
} else {
    SET_LOCAL_ROLE(HEADSET_SEC);
    BT_CPY_BD_ADDR(PRI_BDADDR(), peer_addr);
    BT_CPY_BD_ADDR(SEC_BDADDR(), info->local_addr.addr);
}
```

这里的规则是：

- 如果 `peer_addr` 无效，则 `force_pri = true`，本机按 PRI 工作。
- 如果 `bt_storage_is_pri_dev()` 为真，本机为 PRI，本机地址放入 `PRI_BDADDR()`，对耳地址放入 `SEC_BDADDR()`。
- 否则本机为 SEC，对耳地址放入 `PRI_BDADDR()`，本机地址放入 `SEC_BDADDR()`。

`bt_storage_is_pri_dev()` 在：

`wq-adk/components/bt_service/common/bt_srv_storage.c`

```c
uint32_t bt_storage_is_pri_dev(void)
{
    if (TWS_DEV_PRI == bt_srv_ro_data.tws_role_mode) {
        return 1;
    }
    if (TWS_DEV_SEC == bt_srv_ro_data.tws_role_mode) {
        return 0;
    }
    return bt_srv_general_data.is_pri_dev;
}
```

也就是说，`tws_role_mode` 是更强的角色配置。

## 6. RM 模块如何自动组队

TWS 连接由 RM 模块处理，文件：

`wq-adk/components/bt_service/rm/T_rm_top.c`

先看 `PEER_BDADDR()` 是否有效：

```c
if ((!BT_BD_ADDR_IS_NON_ZERO(PEER_BDADDR()))
    || RM_IS_SINGLE_MODE_PEER_ADDR(PEER_BDADDR())) {
    bt_storage_write_tws_single_mode(STORAGE_TWS_SINGLE_MODE);
    ...
    return SM_MSG_HANDLED;
} else {
    ...
}
```

相关宏在：

`wq-adk/components/bt_service/rm/T_rm.h`

```c
#define RM_SET_SINGLE_MODE_PEER_ADDR(addr_) APPL_SET_ALL_FF_ADDR(addr_)
#define RM_IS_SINGLE_MODE_PEER_ADDR(addr_)  APPL_IS_ALL_FF_ADDR(addr_)
#define RM_IS_VALID_PEER_ADDR(addr_)        APPL_IS_VALID_ADDR(addr_)
```

有效对耳地址存在时，RM 会发起 TWS 链接：

```c
tws_link_connect(PEER_BDADDR(), page_timeout);
```

连接成功后：

```c
bt_service_evt_tws_pair_result(PEER_BDADDR(), true);
bt_storage_write_peer_addr(PEER_BDADDR());
bt_storage_write_tws_single_mode(STORAGE_TWS_DOUBLE_MODE);
RT_STATE_TRANS_TO(dest_id, RM_connected_ID);
```

这一步完成后，设备进入 TWS connected 状态，并把 `tws_single_mode` 记录为双耳模式。

## 7. A-core 对 TWS 配对状态的处理

A-core 应用层文件：

`wq-adk/components/apps/acore/bt/src/app_bt.c`

系统状态计算里，如果 `context->tws_pairing` 为真，则进入：

```c
STATE_WWS_PAIRING
```

关键代码：

```c
if (!wq_bt_is_enabled()) {
    state = STATE_DISABLED;
} else if (context->tws_pairing) {
    state = STATE_WWS_PAIRING;
}
```

主动发起 TWS pairing 命令时：

```c
ret = wq_bt_send_tws_pair_cmd(cmd);
if (!ret) {
    context->tws_pairing = true;
    generate_sys_state();
}
```

收到配对结果：

```c
static void bt_evt_tws_pair_result_handler(const bt_evt_tws_pair_result_t *param)
{
    context->tws_pairing = false;
    if (param->succeed) {
        app_evt_send(EVTSYS_WWS_PAIRED);
    } else {
        app_evt_send(EVTSYS_WWS_PAIR_FAILED);
    }
    generate_sys_state();
}
```

说明：量产预烧 `peer_addr` 后，更主要走的是 RM 自动连接对耳；`STATE_WWS_PAIRING` 更明显用于“没有预置对耳、通过 pairing 流程找对耳”的场景。但手机连接逻辑仍会用这个状态做保护。

## 8. 手机连接为什么会等 TWS 组队

手机回连/配对逻辑在：

`wq-adk/components/apps/acore/bt/src/app_conn.c`

上电处理函数：

```c
void app_conn_handle_bt_power_on(void)
{
    ...
    if (app_bt_get_sys_state(CURRENT_DEVICE_ADDR) == STATE_WWS_PAIRING) {
        connect_reason = CONNECT_REASON_POWER_ON;
        app_bt_set_connectable(false);
        return;
    }
    ...
}
```

也就是说，如果系统正在 TWS 配对，先禁止手机可连接，等 TWS 完成后再继续手机回连或进入手机配对模式。

另外，`peer_addr = FF:FF:FF:FF:FF:FF` 时，`app_conn.c` 中的 `ftm_peer_bdaddr` 会让手机连接流程跳过普通双耳逻辑：

```c
if (bdaddr_is_equal(app_wws_get_peer_addr(), &ftm_peer_bdaddr)) {
    update_discoverable_and_connectable(true, true);
    return;
}
```

## 9. 运行时设置 peer_addr 的接口

除了量产烧录，A-core 也可以通过 RPC 设置 peer address。

A-core API：

`wq-adk/components/bt_rpc/src/acore/wq_bt.c`

```c
WQ_RET wq_bt_set_peer_addr(const BD_ADDR_T *addr)
{
    bt_cmd_tws_set_peer_addr_t param;
    memcpy_s(&param.addr, sizeof(param.addr), addr, sizeof(BD_ADDR_T));
    return wq_send_rpc_cmd(BT_CMD_TWS_SET_PEER_ADDR, &param, sizeof(param));
}
```

B-core RPC 处理：

`wq-adk/components/bt_service/bt_rpc/app_user_cmd.c`

```c
bt_result_t bt_service_tws_set_peer_addr(void *param, uint32_t param_len)
{
    bt_cmd_tws_set_peer_addr_t *info = (bt_cmd_tws_set_peer_addr_t *)param;

    DMSetPeerAddrEvt_T *pe =
        RT_MSG_NEW(DM_SET_PEER_ADDR_SIG,
                   RT_BUILD_ID(MODULE_DM, DM_DEF_ID),
                   0,
                   DMSetPeerAddrEvt_T);
    BT_CPY_BD_ADDR(pe->bd_addr, info->addr.addr);
    pe->set_storage = true;
    RT_MSG_PUT(pe);

    return BT_RESULT_SUCCESS;
}
```

DM 模块收到 `DM_SET_PEER_ADDR_SIG` 后会写入 storage，相关落点可继续看：

`wq-adk/components/bt_service/dm/T_dm_top.c`

## 10. 成功组队后的持久化

BT service 中写 storage 的接口在：

`wq-adk/components/bt_service/common/bt_srv_storage.c`

```c
void bt_storage_write_peer_addr(const uint8_t *peer_addr)
{
    BT_CPY_BD_ADDR(bt_srv_general_data.peer_addr, peer_addr);
    bt_srv_storage_cfg.general_data_dirty = 1;
}

void bt_storage_write_tws_single_mode(uint8_t single_mode)
{
    if (bt_srv_general_data.tws_single_mode != single_mode) {
        bt_srv_general_data.tws_single_mode = single_mode;
        bt_srv_storage_cfg.general_data_dirty = 1;
    }
}
```

实际 flush 通过 `bt_storage_set_writable()` 中的：

```c
bt_service_storage_set(BT_SRV_STORAGE_GENERAL_DATA_ID,
                       (uint8_t *)&bt_srv_general_data,
                       sizeof(bt_general_data_t));
```

所以组队成功后，`peer_addr` 和 `tws_single_mode` 不只存在内存中，也会被标记脏并写回 storage。

## 11. 分场景行为

### 场景 A：量产已指定对耳 MAC

配置：

```text
左耳 local_addr = A
左耳 peer_addr  = B
左耳 tws_role_mode = PRI

右耳 local_addr = B
右耳 peer_addr  = A
右耳 tws_role_mode = SEC
```

启动行为：

```text
bt_storage_load()
  -> 读出 peer_addr
  -> bt_service_power_on_off()
  -> 左耳 SET_LOCAL_ROLE(HEADSET_PRI)
  -> 右耳 SET_LOCAL_ROLE(HEADSET_SEC)
  -> RM 用 PEER_BDADDR 发起 tws_link_connect
  -> 连接成功
  -> bt_storage_write_tws_single_mode(STORAGE_TWS_DOUBLE_MODE)
```

### 场景 B：peer_addr 全 00

配置：

```text
peer_addr = 00:00:00:00:00:00
```

行为：

- `force_pri = true`
- 本机倾向 PRI
- RM 判断没有有效 `PEER_BDADDR()`，写入 `STORAGE_TWS_SINGLE_MODE`
- 如果应用层主动进入 TWS pairing，则会进入 `STATE_WWS_PAIRING`

### 场景 C：peer_addr 全 FF

配置：

```text
peer_addr = FF:FF:FF:FF:FF:FF
```

行为：

- `RM_IS_SINGLE_MODE_PEER_ADDR()` 为真
- RM 认为是 single/factory mode
- 不进行普通 TWS 组队
- 手机连接逻辑里也有 `ftm_peer_bdaddr` 判断，会跳过双耳同步/回连保护

## 12. 快速索引

| 模块 | 文件 | 作用 |
| --- | --- | --- |
| 烧录配置 | `project/a2007/config/*/ro_cfg.xml` | `peer_addr/tws_linkkey/tws_single_mode/tws_role_mode` 配置来源 |
| storage 结构 | `components/bt_service/common/bt_srv_storage.h` | `bt_general_data_t/bt_readonly_data_t` 定义 |
| storage 加载 | `components/bt_service/common/bt_srv_storage.c` | `bt_storage_load()` 加载可写 storage 或 `ro_cfg` 默认值 |
| BT 启动 | `components/bt_service/bt_rpc/app_user_cmd.c` | enable BT 时按 `peer_addr/role` 设置 PRI/SEC 地址 |
| TWS RM | `components/bt_service/rm/T_rm_top.c` | 根据 `PEER_BDADDR()` 自动连接对耳 |
| TWS 宏 | `components/bt_service/rm/T_rm.h` | 有效地址、single mode 地址判断 |
| RPC 设置对耳 | `components/bt_rpc/src/acore/wq_bt.c` | `wq_bt_set_peer_addr()` |
| RPC 命令处理 | `components/bt_service/bt_rpc/app_user_cmd.c` | `BT_CMD_TWS_SET_PEER_ADDR` -> `DM_SET_PEER_ADDR_SIG` |
| A-core 状态 | `components/apps/acore/bt/src/app_bt.c` | `STATE_WWS_PAIRING` 与 TWS pair result |
| 手机连接 | `components/apps/acore/bt/src/app_conn.c` | TWS pairing 时阻塞手机连接 |

## 13. 建议排查日志点

如果后续遇到“烧了对耳 MAC 但不自动组队”，优先看这些日志和变量：

1. `bt_storage_load` 打印的 `peer_addr` 是否为对耳 MAC。
2. `bt_storage_load` 打印的 `tws_role_mode/is_pri/tws_single_mode` 是否符合预期。
3. `bt_service_power_on_off()` 中 `force_pri` 是否异常为 true。
4. `PRI_BDADDR()/SEC_BDADDR()/PEER_BDADDR()` 是否按左右耳正确映射。
5. RM 是否进入 `RM_initialized` 后发现 `PEER_BDADDR()` 无效或全 FF。
6. 是否调用到 `tws_link_connect(PEER_BDADDR(), ...)`。
7. 连接成功后是否执行 `bt_storage_write_tws_single_mode(STORAGE_TWS_DOUBLE_MODE)`。

