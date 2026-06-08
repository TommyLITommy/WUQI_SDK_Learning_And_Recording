# 单线升级（Pogo Pin / 充电盒 UART 升级）流程梳理

本文基于 A2007 工程代码，梳理「充电盒通过 pogo pin 单线 UART 给耳机烧录/升级固件」的完整链路。代码中能找到对应实现，且 A2007 当前构建已启用相关模块。

## 1. 结论先行

「单线升级」在物奇方案里指：**耳机与充电盒之间只有一根通信线（pogo pin）**，UART 的 TX/RX 复用同一 GPIO，半双工通信。

整体分两段：

| 阶段 | 通信内容 | 主要代码 |
| --- | --- | --- |
| 正常运行 | 充电盒 ↔ 耳机：开盖/电量/配对/工厂指令等 | `battery_cmc.c` + `charger_box_none.c` |
| 进入升级 | 充电盒发 `CMD_UPGRADE (0x99)` 或 magic 串 `703XWuQiCli` | `charger_box_none.c` → `switch_to_cli()` |
| 固件烧录 | PC 上位机 ↔ 耳机：CLI / Updater 协议写 Flash | `cli.c` + `cli_common_*.c` 或 `recover/updater` |

A2007 实际编译配置：

- `CONFIG_CHARGER_BOX_NONE=1`，但 `CONFIG_PROJECT_A2002=1`，因此走的是 `charger_box_none.c` 里 **带 UART 通信的 A2002 分支**（不是 stub 版）。
- `CONFIG_CLI_ENABLE=y`、`CONFIG_COMMON_CLI_ENABLE=y` 等，CLI 烧录命令可用。
- pogo pin 对应 **`WQ_GPIO_CHARGER_3_3V`**（7035AX-B 上为 `WQ_GPIO_94`）。

---

## 2. 硬件与 UART 单线模式

### 2.1 Pogo Pin 引脚

充电通信 UART 使用 charger 专用 GPIO，TX/RX 绑定同一 pin：

```48:48:wq-adk/components/battery/src/bat_cmc/battery_cmc.c
#define BATTERY_CMC_UART_UNIQ_PIN WQ_GPIO_CHARGER_3_3V
```

```90:90:wqcore/chipset/hornet/include/resource.h
#define WQ_GPIO_CHARGER_3_3V WQ_GPIO_94
```

初始化时 tx/rx 均指向该 pin，并通过 inner charger 切到 1.8V GPIO 模式：

```298:316:wq-adk/components/battery/src/bat_cmc/battery_cmc.c
void battery_cmc_uart_init(void)
{
    ...
    wq_charger_gpio_enable(true);
    wq_charger_set_gpio_mode(WQ_CHARGER_GPIO_MODE_1P8);
    wq_gpio_close(uniq_pin);

    pin_cfg.tx = uniq_pin;
    pin_cfg.rx = uniq_pin;

    wq_uart_open(BATTERY_CMC_UART_PORT, ...);
}
```

### 2.2 单线 UART 硬件行为

芯片 UART 支持 `uart_share_io_en` 半双工模式，TX/RX 共用 PAD，由 `uart_pad_share_oen` 控制收发方向。详见 `wqcore/docs/auto/sphinx/source/zh_CN/api/driver/uart.rst`。

Recover 分区里的 Updater 也支持显式半双工模式：

```86:90:wqcore/pre-project/recover/components/updater/src/updater_interface_uart.c
    if (update_mode_get() == UPDATER_SEMI_DUPLEX_MODE) {
        gpio_cfg.tx = UPDATER_USE_SEMI_DUPLEX_PIN;
        gpio_cfg.rx = UPDATER_USE_SEMI_DUPLEX_PIN;
        wq_gpio_set_pull_mode(UPDATER_USE_SEMI_DUPLEX_PIN, WQ_GPIO_PULL_UP);
```

---

## 3. 正常运行：充电盒 UART 协议

### 3.1 协议文档

标准协议说明见：

- `wq-adk/components/peripheral/charger_box/src/charger_box_uart.md`

其中工厂指令 **`0x99` 标注为「单线升级」**。

### 3.2 A2007 实际协议头（与标准文档不同）

A2007（A2002 分支）使用的帧头为 **`0xDD 0xAA`**，而非文档中的 `0x55 0xAA`：

```119:120:wq-adk/components/peripheral/charger_box/src/charger_box_none.c
    #define csync_head0                     0xdd
    #define csync_head1                     0xaa
```

帧格式仍为：`Header(2) | Command | L/R | Length | Data | CRC8`。

### 3.3 初始化与收发

`app_charger.c` 启动时注册充电盒回调并打开 CMC UART：

```596:596:wq-adk/components/apps/acore/battery/src/app_charger.c
    charger_box_init(charger_box_callback);
```

```1177:1178:wq-adk/components/peripheral/charger_box/src/charger_box_none.c
        charger_flag_change_register_callback(handle_charge_flag);
        battery_cmc_uart_open(cmc_uart_rx_handler);
```

- UART 端口：`BATTERY_CMC_UART_PORT` = `WQ_UART_PORT_1`
- 波特率：115200
- 接收：`cmc_uart_rx_handler` → `handle_byte` 解析协议或检测 CLI magic 串

---

## 4. 进入 CLI 升级模式（核心切换）

### 4.1 两种触发方式

**方式 A：充电盒发工厂指令 `CMD_UPGRADE (0x99)`**

```888:891:wq-adk/components/peripheral/charger_box/src/charger_box_none.c
        case CMD_UPGRADE:
            os_delay(100);
            switch_to_cli();
            break;
```

**方式 B：逐字节匹配 magic 串 `703XWuQiCli`**

```126:127:wq-adk/components/peripheral/charger_box/src/charger_box_none.c
    static const char cmd_switch_cli[] = {'7', '0', '3', 'X', 'W', 'u', 'Q', 'i', 'C', 'l', 'i'};
```

```988:1000:wq-adk/components/peripheral/charger_box/src/charger_box_none.c
    static inline void handle_byte(uint8_t byte){
        ...
        if (byte == cmd_switch_cli[switch_cli_len]){
            switch_cli_len += 1;
            if (switch_cli_len >= sizeof(cmd_switch_cli)){
                if (!in_cli_mode){
                    switch_to_cli();
                    in_cli_mode = true;
                }
                switch_cli_len = 0;
            }
        }
```

充电盒或 PC 工具经 pogo pin 发送该字符串即可切换，无需完整协议帧。

### 4.2 `switch_to_cli()` 做了什么

```166:175:wq-adk/components/peripheral/charger_box/src/charger_box_none.c
    static void switch_to_cli(void){
        if (in_cli_mode) {
            return;
        }
        in_cli_mode = true;

        battery_cmc_uart_close();
        battery_cmc_uart_switch_to_cli();
    }
```

```263:270:wq-adk/components/battery/src/bat_cmc/battery_cmc.c
void battery_cmc_uart_switch_to_cli(void)
{
    wq_uart_gpio_configuration_t pin_cfg;
    pin_cfg.tx = BATTERY_CMC_UART_UNIQ_PIN;
    pin_cfg.rx = BATTERY_CMC_UART_UNIQ_PIN;
    wq_gpio_close(BATTERY_CMC_UART_UNIQ_PIN);
    wq_uart_gpio_config(BATTERY_CLI_UART_PORT, &pin_cfg);
}
```

关键动作：

1. **关闭** 充电盒协议 UART（`WQ_UART_PORT_1`），停止解析 box 指令。
2. **把 pogo pin 重新挂到 CLI UART**（`BATTERY_CLI_UART_PORT` = `WQ_UART_PORT_0`）。
3. 设置 `in_cli_mode = true`，防止重复切换。

此时 PC 上位机（物奇烧录工具）可通过同一根 pogo pin 与耳机 CLI 通信。

### 4.3 时序概览

```mermaid
sequenceDiagram
    participant Box as 充电盒/上位机
    participant Pin as Pogo Pin (WQ_GPIO_CHARGER_3_3V)
    participant CMC as battery_cmc UART1
    participant BoxProto as charger_box_none
    participant CLI as CLI UART0 + GTP

    Note over Box,BoxProto: 正常运行
    Box->>Pin: 0xDD 0xAA ... 协议帧
    Pin->>CMC: 115200 半双工
    CMC->>BoxProto: cmc_uart_rx_handler

    Note over Box,CLI: 进入升级
    Box->>Pin: CMD_UPGRADE(0x99) 或 "703XWuQiCli"
    BoxProto->>BoxProto: switch_to_cli()
    BoxProto->>CMC: battery_cmc_uart_close()
    BoxProto->>CLI: battery_cmc_uart_switch_to_cli() 复用 pin 到 UART0

    Note over Box,CLI: 烧录
    Box->>Pin: CLI/Updater 协议数据
    Pin->>CLI: generic_transmission → cli_task
    CLI->>CLI: flash/oem/image 写操作
```

---

## 5. CLI 烧录路径（App 运行时）

### 5.1 CLI 框架初始化

系统启动时在 `app_entry()` 中初始化 CLI，并注册 GTP RX 回调：

```317:317:wq-adk/components/apps/acore/entry/entry.c
    cli_init(&cli_config);
```

```457:457:wq-adk/components/cli/src/cli.c
    wq_generic_transmission_register_rx_callback(GENERIC_TRANSMISSION_TID4, cli_rx_callback);
```

GTP UART IO 默认走 `WQ_UART_PORT_0`（`generic_transmission_io_uart.c`）。`switch_to_cli()` 之后，该端口的 GPIO 被切到 pogo pin，上位机数据即可进入 CLI 任务。

### 5.2 常用烧录相关 CLI 命令

| 模块 | 文件 | 能力 |
| --- | --- | --- |
| COMMON | `cli_common_memory.c` | 读/写 RAM、寄存器、Flash 区、CRC 计算 |
| COMMON | `cli_common_oem.c` | OEM 分区读写（量产 MAC、校准数据等） |
| CHARGER | `cli_charger.c` | 关闭 charger GPIO 模式、读 VBUS 等 |

OEM 写入示例（量产烧 peer MAC 等）：

```64:85:wq-adk/components/cli_command/common/cli_common_oem.c
static void cli_oem_write_handler(const uint8_t *buffer, uint32_t bufferlen)
{
    ...
    oem_data_write(tag, data, len);
    cli_interface_msg_response(CLI_MODULEID_COMMON, CLI_MSGID_OEM_WRITE, NULL, 0, ret);
}
```

这与 `TWS_FACTORY_BURN_AUTO_PAIR_FLOW.md` 中描述的「物奇烧录上位机写 ro_cfg / OEM」是同一套 CLI 通道，只是该文档侧重 BT 组队，本文侧重 **物理通道与模式切换**。

### 5.3 OTA 升级（另一条升级路径）

若上位机走 OTA 包升级而非直接写 Flash，固件侧 API 为：

- `wq_ota_begin` → `wq_ota_write_data` → `wq_ota_end` → `wq_ota_commit`

见 `wqcore/docs/manual/ota.md`、`wqcore/components/ota/src/ota.c`。

OTA 完成后可能以 `PM_POWER_ON_REASON_UPGRADE` 重启，关盖时会缩短 BT 关闭延时：

```289:291:wq-adk/components/peripheral/charger_box/src/charger_box_none.c
                    if (app_pm_get_power_on_reason() == PM_POWER_ON_REASON_OTA
                        || (app_pm_get_power_on_reason() == PM_POWER_ON_REASON_UPGRADE)) {
                        os_start_timer(timer_bt_off, 200);
```

---

## 6. Recover / Updater 路径（Boot 阶段烧录）

除 App 内 CLI 外，还有 **Recover 分区 Updater**，用于更底层的镜像烧录（如整包 `IMAGE_WRITE`）。

入口：

```29:53:wqcore/pre-project/recover/bbb/acore/recover.c
void recover_entry(void *arg)
{
    ...
    updater_start(false, 0, CONFIG_RECOVER_INTERFACE);
    updater_chip_jump_to_app();
}
```

Updater 命令包括（节选）：

```34:37:wqcore/pre-project/recover/components/updater/inc/updater_command_id.h
    UPDATER_COMMAND_BOOTMAP_SET = 0x20,
    UPDATER_COMMAND_IMAGE_WRITE,
    UPDATER_COMMAND_IMAGE_ERASE,
    UPDATER_COMMAND_IMAGE_VERIFY,
```

`UPDATER_COMMAND_IMAGE_WRITE` 会擦除 bootmap 对应分区并分片写入 Flash：

```191:231:wqcore/pre-project/recover/components/updater/src/updater_command_memory.c
uint8_t updater_command_memory_image_write(uint8_t *buffer, uint32_t length)
{
    ...
    block->op = UPDATER_FLASH_ERASE_RANGE;
    updater_flash_operate_add(block);
    ...
}
```

当 `updater_start(..., flag=1, ...)` 时进入 **`UPDATER_SEMI_DUPLEX_MODE`**，UART TX/RX 共用单线 pin 通信，与 pogo pin 场景一致。

---

## 7. 相关文件索引

| 类别 | 路径 | 说明 |
| --- | --- | --- |
| 协议文档 | `wq-adk/components/peripheral/charger_box/src/charger_box_uart.md` | 充电盒协议，`0x99`=单线升级 |
| Box 实现 (A2007) | `wq-adk/components/peripheral/charger_box/src/charger_box_none.c` | `CMD_UPGRADE`、`switch_to_cli`、协议解析 |
| Box 实现 (标准) | `wq-adk/components/peripheral/charger_box/src/charger_box_uart.c` | 头 `0x55 0xAA`，逻辑类似 |
| CMC UART | `wq-adk/components/battery/src/bat_cmc/battery_cmc.c` | pogo pin UART 开/关/切 CLI |
| CMC 头文件 | `wq-adk/components/battery/inc/battery_cmc.h` | API 声明 |
| 充电 App | `wq-adk/components/apps/acore/battery/src/app_charger.c` | `charger_box_init` |
| CLI 框架 | `wq-adk/components/cli/src/cli.c` | CLI 任务与 GTP 对接 |
| CLI 烧录命令 | `wq-adk/components/cli_command/common/` | memory/oem 等 |
| GTP UART IO | `wqcore/components/generic_transmission/src/io_methods/generic_transmission_io_uart.c` | UART0 日志/CLI 通道 |
| Recover Updater | `wqcore/pre-project/recover/components/updater/` | Boot 阶段镜像烧录 |
| UART 单线说明 | `wqcore/docs/auto/sphinx/source/zh_CN/api/driver/uart.rst` | 半双工硬件原理 |
| OTA 说明 | `wqcore/docs/manual/ota.md` | OTA 包升级流程 |
| 量产烧 MAC | `wq-adk/project/a2007/docs/TWS_FACTORY_BURN_AUTO_PAIR_FLOW.md` | 上位机写 OEM 后的 BT 行为 |

---

## 8. 调试与排查建议

1. **确认是否已进入 CLI 模式**：抓 log 是否有 `switch_to_cli`（`DBGLOG_CHARGER_BOX_NONE_DBG`）。
2. **确认协议头**：A2007 用 `0xDD 0xAA`，不要按文档 `0x55 0xAA` 组包。
3. **确认 magic 串**：`703XWuQiCli`（11 字节，大小写敏感）。
4. **确认 pin 切换**：升级前 UART1 收 box 协议；升级后 UART0 收 CLI。若上位机仍发 box 帧会被忽略。
5. **Recover vs App CLI**：Recover Updater 在 boot 早期；App CLI 需耳机已启动且收到 `0x99` 或 magic 串。工具需匹配当前阶段协议。
6. **关盖/入盒行为**：升级过程中关盖可能触发 `CHARGER_EVT_BOX_CLOSE`、`timer_bt_off` 等，注意 `PM_POWER_ON_REASON_UPGRADE` 分支的时序差异（bug 50197 相关逻辑）。

---

## 9. 与标准 `charger_box_uart.c` 的差异

| 项目 | `charger_box_uart.c` | A2007 `charger_box_none.c` (A2002) |
| --- | --- | --- |
| 帧头 | `0x55 0xAA` | `0xDD 0xAA` |
| CRC 算法 | 多项式 0xD8 位运算 | 查表 `gcrc_tab_x8_x5_x4_1` |
| 配置宏 | `CONFIG_CHARGER_BOX_UART` | `CONFIG_CHARGER_BOX_NONE` + `CONFIG_PROJECT_A2002` |
| 升级入口 | 相同：`CMD_UPGRADE` / `703XWuQiCli` / `switch_to_cli` | 相同 |

两套实现的 **单线升级切换逻辑一致**，差异主要在 box 日常通信协议格式。
