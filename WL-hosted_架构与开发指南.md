# WL-hosted 架构与开发指南

> 状态：架构草案（Draft）  
> 目标读者：WL-hosted 维护者、Host/Coprocessor 适配开发者、示例工程开发者、自动化 Agent  
> 适用范围：MCU/POSIX Host 通过 SDIO、SPI、UART 或 USB 使用可编程无线 SoC 提供 Wi-Fi、Bluetooth 等能力

---

## 1. 文档目的

WL-hosted 是一个面向多 MCU、多无线 SoC 和多软件生态的 Hosted Wireless 框架。其目标不是复制某一家芯片厂商的 Host Driver，也不是将 AT 命令简单替换为二进制 RPC，而是提供一套可移植、可版本化、可恢复、可扩展的“分布式无线驱动”架构。

WL-hosted 的基本模型是：

```text
Host MCU
  ├─ 运行应用、TCP/IP 网络栈和上层协议
  ├─ 通过统一 Host API 控制无线功能
  └─ 通过 SDIO/SPI/UART/USB 与 Coprocessor 通信

Wireless Coprocessor
  ├─ 运行厂商 Wi-Fi/Bluetooth 驱动和无线协议栈固件
  ├─ 执行 Host 发起的控制 RPC
  ├─ 转发 Ethernet、Bluetooth HCI 等数据
  └─ 以预编译 Release 固件为主要交付形式
```

对普通应用开发者而言，开发中心应始终位于 Host：

```text
选择 Host 开发板
选择 Coprocessor 固件 Profile
烧录官方 Coprocessor Release 固件
开发和调试 Host 应用
```

普通用户不应被要求安装 Coprocessor 厂商 SDK、编译 Coprocessor 工程或理解其分区表和内部实现。

---

## 2. 术语与命名

### 2.1 Host

运行产品主应用和 TCP/IP 网络栈的 MCU 或处理器。Host 通常也是 SDIO Host 或 SPI Master，但“Host”描述的是系统角色，不等同于具体总线角色。

### 2.2 Coprocessor

提供 Wi-Fi、Bluetooth、OpenThread 或其他无线能力的可编程 SoC。Coprocessor 通常运行 WL-hosted 固件，并以稳定的协议接口暴露无线功能。

仓库名和公共文档中统一使用缩写 `coproc`：

```text
wl-hosted-coproc-core
wl-hosted-coproc-esp-idf
```

### 2.3 Master / Slave

只在描述具体总线角色时使用，例如：

```text
SPI Master / SPI Slave
SDIO Host / SDIO Device
```

禁止使用 Master/Slave 表示 WL-hosted 的系统角色。

### 2.4 Adapter

将 Core 接入某个 SDK、RTOS、网络栈、芯片系列或无线厂商 API 的适配层。

### 2.5 Board

单块开发板或产品板的定义。Board 只描述本侧硬件能力，不描述对端板卡。

### 2.6 Firmware Profile

一个可直接烧录的 Coprocessor 固件变体。Profile 由芯片、板卡/模组、传输方式、功能集合、Flash 布局和安全策略共同决定。

### 2.7 Preset

Host 侧的已验证组合配置，用于绑定：

```text
Host Board + Host Adapter + Transport 参数 + Coprocessor Firmware Profile
```

### 2.8 Example

面向 Host 应用开发者的示例应用，例如 Wi-Fi Station、iperf、SoftAP 或 Bluetooth HCI。Example 描述“应用做什么”，不描述具体 Coprocessor 源码如何构建。

---

## 3. 核心设计原则

### 3.1 控制面与数据面分离

WL-hosted 必须将低频控制消息和高频数据流分开：

```text
控制面：RPC
  - 初始化
  - 扫描
  - 连接/断开
  - 国家码
  - 功率模式
  - 统计信息
  - 能力协商
  - IO/GPIO 配置、读写
  - ADC 读取
  - KV 读写擦除
  - 设备信息读取
  - 用户小消息透传

数据面：Raw Stream / Frame
  - Ethernet frame
  - Bluetooth HCI
  - OTA 数据
  - 日志和诊断数据
  - 用户透传大消息
```

禁止将每个 Ethernet 包、HCI 包或固件块包装成通用 RPC。

### 3.2 上层 API 与厂商 API 解耦

WL-hosted 公共 API 和 Wire Protocol 不得直接暴露以下内容：

- `esp_wifi_*` 类型或错误码；
- Infineon WHD 内部结构；
- Realtek、Beken 或其他厂商私有结构；
- 某个 RTOS 的句柄类型；
- 某个网络栈的 packet 类型；
- 某个 MCU HAL 的 DMA 或总线类型。

标准功能通过 WL-hosted 自有类型描述；厂商特有能力通过显式 Vendor Extension 提供。

### 3.3 协议与传输解耦

RPC、Channel 和 Service 层不应知道底层使用 SDIO、SPI、UART 还是 USB。不同传输只负责可靠地交换 WL-hosted Frame。

### 3.4 RTOS 与 OSAL

Host 和 Coprocessor 都强制要求可用的 RTOS 语义，不支持裸机主循环或由应用手工 Poll Core。POSIX 模拟器视为一种 RTOS 运行环境，由 pthread、condition variable 和 monotonic clock 等能力实现 OSAL。

Core 不得直接调用 FreeRTOS、Zephyr、RT-Thread、pthread 或某个厂商 OS API，必须通过 WL-hosted OSAL 使用：

- Task/Thread；
- Mutex/Semaphore/Event；
- Queue；
- Timer 和 monotonic time；
- Sleep/Yield；
- ISR-safe 通知与上下文判断。

OSAL 只统一 Core 需要的最小 OS 语义，不封装网络栈、总线 HAL 或厂商无线 API。Task 数量、栈大小、优先级和队列深度由 Adapter/Profile 显式配置，并保持资源上界。

### 3.5 静态资源优先

核心层默认以有界、可预测的资源模型为目标：

- 固定最大 pending RPC 数量；
- 固定或可配置 buffer pool；
- 有上限的 protobuf 字段；
- 明确的 queue/credit 上限；
- 不依赖无界动态分配。

允许 Adapter 在资源充足的平台上使用动态内存，但公共协议和 Core 不得以动态内存为正确运行的前提。

### 3.6 Host 优先的开发体验

普通用户主要使用 Host Adapter 仓库中的 Example 和 Preset。Coprocessor 以预编译固件交付，源代码仓库主要面向维护者和新的无线 SoC 适配者。

### 3.7 可恢复性是基础功能

链路重建、Coprocessor 重启、总线超时、RPC 超时、buffer 耗尽和版本不兼容必须从第一版协议中考虑，而不是在后期通过整机重启临时补救。

---

## 4. 非目标

WL-hosted 初期不追求：

- 兼容所有厂商原生 Wi-Fi API；
- 将所有无线 SoC 抽象成完全相同的高级特性集合；
- 在第一版实现所有 Wi-Fi Enterprise、CSI、FTM、TWT 和监控模式；
- 让普通 Host 应用开发者编译 Coprocessor 固件；
- 为所有理论上的 Host × Coprocessor 组合提供官方支持；
- 在协议 1.0 前保持所有实验性接口永久兼容；
- 让 protobuf 承担大数据流传输；
- 让 Core 直接管理具体开发板引脚。

---

## 5. 总体架构

```text
┌────────────────────────────────────────────────────────────┐
│ Host Application                                           │
│  sockets / TCP/IP / TLS / MQTT / HTTP / product logic     │
├────────────────────────────────────────────────────────────┤
│ WL-hosted Host Public API                                  │
│  wifi / bluetooth / diagnostics / coproc management       │
├────────────────────────────────────────────────────────────┤
│ Host Core                                                  │
│  state machine / RPC client / mux / credits / recovery     │
├────────────────────────────────────────────────────────────┤
│ Host Adapter                                               │
│  OSAL / netif / buffers / board / transport HAL          │
└───────────────────────┬────────────────────────────────────┘
                        │ WL-hosted Wire Protocol
┌───────────────────────▼────────────────────────────────────┐
│ Coprocessor Adapter                                        │
│  bus device HAL / vendor Wi-Fi API / BT controller / OTA  │
├────────────────────────────────────────────────────────────┤
│ Coprocessor Core                                           │
│  link server / RPC dispatch / mux / credits / recovery     │
├────────────────────────────────────────────────────────────┤
│ Vendor Wireless SDK                                        │
│  Wi-Fi firmware / MAC / PHY / Bluetooth controller         │
└────────────────────────────────────────────────────────────┘
```

### 5.1 平面划分

```text
Link Plane
  - 会话建立
  - 协议协商
  - 心跳
  - Credit
  - Reset/Recovery

Control Plane
  - Wi-Fi RPC
  - Bluetooth 控制
  - OTA 控制
  - 诊断命令

Data Plane
  - STA Ethernet
  - AP Ethernet
  - Bluetooth HCI
  - OTA Stream
  - Log Stream
```

---

## 6. 仓库规划

推荐的基础仓库如下：

```text
wl-hosted-protocol
wl-hosted-host-core
wl-hosted-coproc-core

wl-hosted-host-<ecosystem>
wl-hosted-coproc-<vendor-sdk>

wl-hosted-firmware-catalog
wl-hosted-tools
```

示例：

```text
wl-hosted-protocol
wl-hosted-host-core
wl-hosted-coproc-core

wl-hosted-host-hpm-sdk
wl-hosted-host-stm32cube
wl-hosted-host-zephyr
wl-hosted-host-esp-idf

wl-hosted-coproc-esp-idf
wl-hosted-coproc-beken-sdk
wl-hosted-coproc-realtek-sdk

wl-hosted-host-macos-sim
wl-hosted-coproc-macos-sim
wl-hosted-macos-sim-manager

wl-hosted-firmware-catalog
wl-hosted-tools
```

### 6.1 仓库拆分原则

Adapter 仓库按 SDK 或生态拆分，而不是按单颗芯片拆分：

```text
推荐：wl-hosted-host-stm32cube
不推荐：wl-hosted-host-stm32h743
不推荐：wl-hosted-host-stm32h747
```

同一 Adapter 仓库可以包含多个芯片系列和开发板定义。只有在 SDK、许可证、构建系统、维护团队或发布周期显著不同的情况下才拆新仓库。

### 6.2 依赖方向

```text
                    wl-hosted-protocol
                       ▲          ▲
                       │          │
        wl-hosted-host-core      wl-hosted-coproc-core
                       ▲          ▲
                       │          │
          wl-hosted-host-*      wl-hosted-coproc-*
```

允许：

```text
host adapter -> host core -> protocol
coproc adapter -> coproc core -> protocol
```

禁止：

```text
host core -> 某个 Host Adapter
coproc core -> 某个 Coprocessor Adapter
host adapter -> coproc adapter
coproc adapter -> host adapter
protocol -> RTOS/HAL/厂商 SDK
```

---

## 7. 各仓库职责

### 7.1 `wl-hosted-protocol`

这是最稳定、审查最严格的仓库，负责定义“线上传什么”。

推荐目录：

```text
wl-hosted-protocol/
├── spec/
│   ├── architecture.md
│   ├── wire-format.md
│   ├── compatibility.md
│   ├── error-model.md
│   ├── security.md
│   ├── services/
│   │   ├── link.md
│   │   ├── wifi.md
│   │   ├── bluetooth.md
│   │   ├── ota.md
│   │   ├── diagnostics.md
│   │   ├── io.md
│   │   ├── adc.md
│   │   ├── kv.md
│   │   ├── device-info.md
│   │   └── user-passthrough.md
│   └── transports/
│       ├── sdio.md
│       ├── spi.md
│       ├── uart.md
│       └── usb.md
├── proto/
│   ├── link.proto
│   ├── wifi.proto
│   ├── bluetooth.proto
│   ├── ota.proto
│   ├── diagnostics.proto
│   ├── io.proto
│   ├── adc.proto
│   ├── kv.proto
│   ├── device_info.proto
│   ├── user_passthrough.proto
│   └── nanopb.options
├── include/wlh/
│   ├── wire.h
│   ├── ids.h
│   ├── endian.h
│   └── errors.h
├── src/
│   ├── frame_encode.c
│   └── frame_decode.c
├── generated/
│   └── nanopb/
├── tests/
│   ├── vectors/
│   ├── malformed/
│   └── compatibility/
└── tools/
    └── generate.py
```

必须包含：

- Wire Format；
- Channel ID、Service ID 和 Method ID 分配；
- protobuf schema；
- 协议版本规则；
- 错误码域；
- Capability Negotiation；
- Transport Binding；
- Golden Test Vector；
- 生成器版本锁定。

不得包含：

- 线程创建；
- Board 引脚；
- 某个 Wi-Fi 厂商 API；
- 某个 MCU HAL；
- 产品应用代码。

### 7.2 `wl-hosted-host-core`

负责定义 Host 侧状态机和公共运行时，回答“Host 如何工作”。

主要职责：

- 链路初始化与能力协商；
- Session 管理；
- Channel Mux；
- RPC Client；
- pending request 匹配；
- 异步 Event 分发；
- TX 调度；
- Credit 流控；
- 超时与取消；
- Coprocessor 重启检测；
- 链路恢复；
- 统计和诊断；
- Host 公共 API 的平台无关部分。

不得直接依赖：

- FreeRTOS、Zephyr、RT-Thread；
- lwIP、NetX Duo、Zephyr Net；
- STM32 HAL、HPM SDK、ESP-IDF；
- 某个 Coprocessor Adapter。

### 7.3 `wl-hosted-coproc-core`

负责 Coprocessor 侧的通用状态机和 Service Dispatcher。

主要职责：

- 链路服务端状态机；
- RPC Dispatcher；
- Service Registry；
- Capability Provider；
- Ethernet/HCI/OTA Channel 分发；
- Host Reset 和 Session 重建；
- Queue、Credit 和调度；
- 公共诊断统计；
- Coprocessor OTA 框架。

不得直接依赖某一家无线厂商 SDK。

### 7.4 `wl-hosted-host-<ecosystem>`

这是普通应用开发者的主要入口。

主要包含：

- Host Core 的 OS/SDK 适配；
- 网络栈适配；
- buffer、cache 和 DMA 适配；
- SDIO/SPI/UART/USB Host 驱动适配；
- Host OSAL 实现与资源配置；
- Host Board 定义；
- Host Example；
- 已验证的 Coprocessor Preset；
- 下载和烧录 Coprocessor Release 固件的工具。

### 7.5 `wl-hosted-coproc-<vendor-sdk>`

面向无线 SoC 适配和固件维护者。

主要包含：

- Coprocessor Core 的 SDK/OS 适配；
- 厂商 Wi-Fi/Bluetooth API 适配；
- SDIO Device、SPI Slave、UART、USB Device 适配；
- Coprocessor OSAL 实现与资源配置；
- 芯片和板卡 Profile；
- 分区与安全配置；
- Release 构建 CI；
- Factory/OTA 固件产物生成。

普通 Host 应用开发者通常不需要 clone 此仓库。

### 7.6 `wl-hosted-firmware-catalog`

提供所有官方 Coprocessor Firmware Profile 的中央索引。

主要包含：

- Profile 元数据；
- 固件版本；
- 下载位置；
- SHA-256；
- 签名；
- 协议兼容范围；
- 功能列表；
- 已知问题；
- Stable/Beta/Nightly 发布通道。

二进制文件优先存放在对应 Coprocessor Adapter 的 Release 页面，Catalog 只保存元数据和下载引用，避免 Git 历史被大量固件膨胀。

### 7.7 `wl-hosted-tools`

包含跨平台工具和测试基础设施：

- 固件下载和烧录；
- Wire Trace Decoder；
- Wireshark Dissector；
- POSIX Simulator；
- Fuzz Harness；
- 协议代码生成；
- HIL 测试控制器；
- Release Manifest 校验器。

---

## 8. Host Example、Board 与 Preset

### 8.1 Example 只放在 Host Adapter

Example 面向应用开发者，因此放在 Host Adapter 仓库：

```text
wl-hosted-host-hpm-sdk/
├── examples/
│   ├── wifi_station/
│   ├── wifi_iperf/
│   ├── wifi_softap/
│   ├── ble_hci/
│   ├── coproc_ota/
│   └── link_diagnostics/
```

Coprocessor Adapter 不需要提供面向普通用户的对称 Example。它应提供 Profile 构建配置、内部测试和 Release 固件。

### 8.2 三个概念必须分离

```text
Example：应用逻辑，例如连接 AP 并运行 iperf
Board：Host 开发板能力，例如 HPM6750EVK
Preset：Host Board 与某个 Coprocessor 固件的已验证组合
```

完整运行配置为：

```text
Example + Host Board + Coprocessor Preset
```

### 8.3 推荐 Host Adapter 目录

```text
wl-hosted-host-hpm-sdk/
├── include/
├── src/
├── boards/
│   ├── hpm6750evk/
│   │   ├── board.c
│   │   ├── board.h
│   │   ├── board.yml
│   │   └── default.conf
│   └── hpm5301evklite/
├── examples/
│   ├── wifi_station/
│   ├── wifi_iperf/
│   └── coproc_ota/
├── presets/
│   ├── hpm6750evk__esp32c6-devkitc__sdio4.yml
│   ├── hpm6750evk__esp32c5-devkitc__sdio4.yml
│   └── hpm5301evklite__esp32c3-devkitm__spi.yml
└── tools/
    ├── configure.py
    ├── fetch_coproc_firmware.py
    └── flash_coproc.py
```

### 8.4 Preset 示例

```yaml
schema_version: 1

id: hpm6750evk__esp32c6-devkitc__sdio4
support_tier: tier-1

host:
  board: hpm6750evk
  adapter: hpm-sdk

coprocessor:
  firmware_profile: espressif.esp32c6.devkitc.sdio4-wifi-ble
  recommended_version: 0.3.2

transport:
  type: sdio
  bus_width: 4
  clock_hz: 40000000
  block_size: 512

host_pins:
  clk: PC12
  cmd: PC13
  d0: PC14
  d1: PC15
  d2: PC16
  d3: PC17
  reset: PA08

features:
  wifi_sta: true
  wifi_ap: true
  bluetooth_hci: true
  coprocessor_ota: true
```

Preset 只描述 Host 侧需要知道的内容，不包含 Coprocessor 源码或其完整构建工程。

### 8.5 不复制 Example

禁止创建：

```text
examples/hpm6750-esp32c6-station/
examples/hpm6750-esp32c5-station/
examples/hpm5301-esp32c3-station/
```

应复用一个 `wifi_station`，由 Board 和 Preset 注入配置。

---

## 9. Coprocessor 固件交付模型

### 9.1 Firmware Profile 的组成

Profile ID 不能只包含芯片型号。同一芯片可能存在不同总线、功能和安全配置。

推荐 ID：

```text
espressif.esp32c6.devkitc.sdio4-wifi
espressif.esp32c6.devkitc.sdio4-wifi-ble
espressif.esp32c6.wroom1.spi-wifi
espressif.esp32s3.coreboard.usb-wifi
beken.bk7258.devkit.sdio4-wifi
```

Profile 至少由以下因素决定：

- Vendor；
- Chip；
- Board/Module；
- Transport；
- Bus Width；
- Pin Mapping；
- Flash Size；
- Feature Set；
- Partition Layout；
- Secure Boot/Encryption 策略；
- 协议兼容范围。

### 9.2 Release 产物

每个正式 Profile 至少发布：

```text
<profile>-v<version>.factory.bin
<profile>-v<version>.ota.bin
<profile>-v<version>.manifest.json
<profile>-v<version>.sha256
<profile>-v<version>.sig
```

含义：

| 文件 | 用途 |
|---|---|
| `factory.bin` | 首次烧录，尽量包含 bootloader、分区表和应用 |
| `ota.bin` | 由 Host 或其他升级路径更新 Coprocessor |
| `manifest.json` | 烧录地址、版本、协议、功能、兼容性和校验信息 |
| `.sha256` | 完整性校验 |
| `.sig` | Release 签名 |

如果芯片不支持合并 Factory Image，Manifest 必须列出所有烧录段及地址。

### 9.3 Profile Manifest 示例

```json
{
  "schema_version": 1,
  "profile_id": "espressif.esp32c6.devkitc.sdio4-wifi-ble",
  "firmware_version": "0.3.2",
  "build_id": "82f9c3a1",
  "release_channel": "stable",
  "protocol": {
    "major": 1,
    "min_minor": 2,
    "max_minor": 5
  },
  "transport": {
    "type": "sdio",
    "bus_width": 4,
    "max_clock_hz": 40000000,
    "block_size": 512
  },
  "features": [
    "wifi-sta",
    "wifi-ap",
    "bluetooth-hci",
    "coproc-ota",
    "diagnostics"
  ],
  "artifacts": {
    "factory": {
      "file": "factory.bin",
      "sha256": "..."
    },
    "ota": {
      "file": "ota.bin",
      "sha256": "..."
    }
  }
}
```

### 9.4 理想用户流程

```text
选择 Host Adapter 中的 Example、Board 和 Preset
        ↓
使用 Adapter 自身构建系统生成 Host 配置
        ↓
运行 Adapter 提供的固件获取/烧录脚本
        ↓
使用对应 SDK 的原生构建与烧录流程构建 Host
```

项目不提供统一的 `wlh` 命令行工具。具体命令由 Host Adapter 按其生态给出，例如 CMake Preset、ESP-IDF Component 或厂商 SDK 工程脚本。

用户不应被要求：

- 安装 ESP-IDF/Beken SDK；
- 编译 Coprocessor；
- 手工生成分区表；
- 手工匹配 protobuf 文件；
- 自己判断协议兼容性。

### 9.5 Host 管理 Coprocessor 固件

后续支持三种模式：

```text
EXTERNAL
  开发阶段由外部工具烧录 Coprocessor

HOST_OTA
  Host 通过 WL-hosted OTA Service 更新 Coprocessor

EMBEDDED
  Host 固件内嵌 Coprocessor Recovery/Factory Image
```

启动时可执行：

```text
查询 Coprocessor 固件与协议版本
        ↓
兼容：正常启动
不兼容且允许自动升级：执行 Coprocessor OTA
        ↓
重启 Coprocessor并重新协商
```

固件版本不应替代运行时协议协商。

---

## 10. Wire Protocol 设计

### 10.1 三层结构

```text
Frame Layer
  通用边界、Channel、长度、Session、序号、校验

Channel Layer
  Link/RPC/Ethernet/HCI/OTA/Log 的复用与流控

Service Layer
  RPC Request/Response/Event 和具体 protobuf payload
```

### 10.2 Frame Header 原则

Wire Format 必须逐字节定义，不允许将 C struct 直接作为 ABI。

禁止：

- C bitfield；
- 未明确端序的整数；
- 依赖 `__attribute__((packed))`；
- 依赖编译器 enum 大小；
- 通过 `memcpy()` 将内部结构直接发送上线。

推荐字段：

```text
magic
protocol_major
header_size
channel
flags
payload_size
session_id
sequence
header_crc32c
payload_crc32c（可选）
```

下面仅为逻辑表示，不是允许直接序列化的 C ABI：

```c
struct wlh_frame_header_view {
    uint16_t magic;
    uint8_t protocol_major;
    uint8_t header_size;
    uint8_t channel;
    uint8_t flags;
    uint16_t payload_size;
    uint32_t session_id;
    uint32_t sequence;
    uint32_t header_crc32c;
};
```

编码必须使用显式函数：

```c
wlh_put_le16(buf + OFFSET_MAGIC, WLH_MAGIC);
wlh_put_le32(buf + OFFSET_SESSION_ID, session_id);
```

### 10.3 Channel 建议

```text
0x00 LINK_CONTROL
0x01 WIFI_RPC
0x02 ETHERNET_STA
0x03 ETHERNET_AP
0x04 BLUETOOTH_HCI
0x05 OTA_STREAM
0x06 DIAGNOSTICS
0x07 LOG_STREAM
0x08 USER_PASSTHROUGH
0x09~0x3F Reserved Standard Channels
0x40~0x7F Experimental Channels
0x80~0xFF Vendor/Private Channels
```

具体编号由 Protocol 仓库统一维护，已发布编号不得复用。

### 10.4 RPC Envelope

RPC Header 使用固定格式，protobuf 只编码具体参数：

```text
service_id
method_id
request_id
kind: request / response / event
flags
payload_size
status_domain
status_code
protobuf payload
```

逻辑表示：

```c
struct wlh_rpc_header_view {
    uint16_t service_id;
    uint16_t method_id;
    uint32_t request_id;
    uint8_t kind;
    uint8_t flags;
    uint16_t payload_size;
    uint16_t status_domain;
    int16_t status_code;
};
```

不得创建一个包含全部系统 RPC 的巨型 `oneof`，也不得让 protobuf field number 同时承担全局 Method ID。

### 10.5 Service 拆分

推荐：

```text
link.proto
wifi.proto
bluetooth.proto
ota.proto
diagnostics.proto
io.proto
adc.proto
kv.proto
device_info.proto
user_passthrough.proto
```

Wi-Fi 初期标准 Service：

```text
initialize
deinitialize
scan_start
scan_cancel
connect
disconnect
get_link_info
set_country
get_country
set_power_mode
get_statistics
start_ap
stop_ap
```

标准但可选的设备能力 Service：

```text
IO Service
  io_configure
  io_read
  io_write

ADC Service
  adc_read

KV Service
  kv_read
  kv_write
  kv_erase

Device Information Service
  get_device_info

User Passthrough Service
  user_message_send
```

这些 Service 不是 Mandatory Service。Coprocessor 必须通过能力协商声明支持的 Service 及版本；Host 在未声明支持时不得调用对应 Method。

### 10.6 系统能力 Service

#### IO Service

IO 接口只使用逻辑 `pin_id`，由 Board/Profile 映射到实际 MCU 引脚，不在 Wire Protocol 中传输厂商引脚编号。

`io_configure` 字段：

```text
pin_id
mode: INPUT / OUTPUT / OPEN_DRAIN
pull: NONE / UP / DOWN
initial_level
```

`io_read` 返回引脚电平、当前 mode 和 pull配置。`io_write` 只允许配置为 OUTPUT 或 OPEN_DRAIN 的引脚执行。第一版不定义 GPIO 中断事件。

#### ADC Service

`adc_read` 输入逻辑 `pin_id`，返回 `millivolts` 。ADC 分辨率、参考电压和校准实现属于 Coprocessor Adapter 内部。引脚不支持 ADC、引脚非法或校准失败时必须返回明确错误。

#### KV Service

KV 使用 UTF-8 字符串 `key` 和 `value`，提供：

```text
kv_read(key)
kv_write(key, value)
kv_erase(key)
```

`kv_write` 和 `kv_erase` 是单操作原子提交，write 成功表示已达到协议要求的持久化语义。最大 key/value 长度、总容量和剩余容量由能力协商声明。不定义事务、批量提交和枚举接口。删除不存在的 Key 返回 `NOT_FOUND`。

#### Device Information Service

`get_device_info` 返回结构化字段：

```text
vendor
mcu_model
uid: bytes
```

这些字段不使用 JSON，也不传输厂商内部 C 结构体。

#### User Passthrough Service

RPC 形式的 `user_message_send` 包含：

```text
endpoint_id
message_type
flags
payload
```

该 Method 为 send-only，允许对端通过异步 response/event 返回可选结果。高频或大消息使用 `USER_PASSTHROUGH` Raw Stream Channel，消息包含相同的 endpoint/type/flags 和 payload，支持分片、聚合和 Per-Channel Credit。一次传输可包含多个消息，消息也可跨多次传输，用户 payload 不由 WL-hosted 解释。

以下是逻辑 protobuf 字段定义，实际 `.proto` 文件必须在 Protocol 仓库中作为唯一 Schema 来源：

```protobuf
message IoConfigureRequest {
  uint32 pin_id = 1;
  IoMode mode = 2;
  IoPull pull = 3;
  bool initial_level = 4;
}

message IoReadRequest { uint32 pin_id = 1; }
message IoReadResponse {
  uint32 pin_id = 1;
  bool level = 2;
  IoMode mode = 3;
  IoPull pull = 4;
}
message IoWriteRequest {
  uint32 pin_id = 1;
  bool level = 2;
}
message AdcReadRequest { uint32 pin_id = 1; }
message AdcReadResponse {
  uint32 pin_id = 1;
  uint32 millivolts = 2;
}
message KvReadRequest { string key = 1; }
message KvReadResponse { string value = 1; }
message KvWriteRequest {
  string key = 1;
  string value = 2;
}
message KvEraseRequest { string key = 1; }
message DeviceInfoResponse {
  string vendor = 1;
  string mcu_model = 2;
  bytes uid = 3;
  string board_profile = 4;
}
message UserMessageSend {
  uint32 endpoint_id = 1;
  uint32 message_type = 2;
  uint32 flags = 3;
  bytes payload = 4;
}
```

`IoMode`、`IoPull` 的枚举值、Service ID、Method ID、字段上限和 Raw Stream 消息头必须在 Protocol 仓库中统一分配，已发布编号不得复用。

高级厂商能力通过 Vendor Service：

```text
vendor_id
vendor_service_id
vendor_method_id
version
payload
```

Vendor Payload 仍需明确版本和 Wire Schema，禁止发送厂商内部 C struct。

---

## 11. Protobuf 与 nanopb 规则

### 11.1 默认选择

WL-hosted 的协议仓库以 `.proto` 为唯一 schema 来源，Host Core 和 Coprocessor Core 初期统一使用 nanopb。

原因：

- 适合资源受限 MCU；
- 可静态分配；
- 字段上限明确；
- 代码体积和 RAM 更可控；
- 支持流式编码/解码；
- 两端仍保持标准 protobuf Wire Compatibility。

未来资源充足的平台可以增加 protobuf-c 或其他标准 protobuf backend，但不得改变 Wire Protocol。

### 11.2 Schema 限制

必须：

- 使用 proto3；
- string/bytes 设置 `max_size`；
- repeated 设置 `max_count`；
- 消息大小有明确上限；
- 删除字段后使用 `reserved`；
- 未知字段可安全跳过；
- 大数据使用 Stream Channel。

禁止：

- `map`；
- 无上限 repeated；
- 无上限 bytes/string；
- 用 protobuf 传输整包 Ethernet；
- 用 protobuf 传输大型固件；
- 公共 API 直接暴露 nanopb 生成类型。

### 11.3 公共类型与生成类型隔离

公共 API：

```c
struct wlh_wifi_connect_params {
    uint8_t ssid[32];
    uint8_t ssid_len;
    uint8_t passphrase[64];
    uint8_t passphrase_len;
    uint32_t timeout_ms;
};
```

内部编码层负责在公共类型与 nanopb 类型之间转换。这样未来替换 codec 时不会破坏应用 API。

### 11.4 生成器锁定

Protocol 仓库必须固定：

```text
protoc 版本
nanopb 版本
nanopb.options
生成命令
生成结果校验值
```

CI 必须验证生成文件与 schema 一致。

---

## 12. Core 执行模型

### 12.1 Host Core 状态机

建议状态：

```text
UNINITIALIZED
TRANSPORT_STARTING
WAITING_FOR_PEER
NEGOTIATING
READY
CONGESTED
RECOVERING
FAILED
STOPPING
```

### 12.2 Coprocessor Core 状态机

```text
BOOTING
TRANSPORT_READY
WAITING_FOR_HOST
NEGOTIATING
READY
RECOVERING
UPDATING
FAILED
```

### 12.3 RTOS 执行模型

Host Core 和 Coprocessor Core 由 OSAL Task 驱动。参考执行模型至少包含：

```text
Transport RX/ISR
  -> ISR-safe 通知
  -> RX Task 解析 Frame 并投递事件

TX/RPC Task
  -> 调度 Channel、Credit 和 Timeout

Application Worker/Executor
  -> 执行允许阻塞的用户 Callback
```

Adapter 可以在满足上述语义的前提下合并 Task，但必须证明 RX 不会被用户 Callback 或慢速总线操作长时间阻塞。POSIX 实现使用 pthread 运行相同执行模型，不设置特殊的 Poll-only Core 路径。

### 12.4 RPC 请求匹配

必须使用：

```text
pending[request_id] -> waiter/callback/context/deadline
```

收到响应时：

```text
查找 request_id
验证 session_id、service_id 和 method_id
移除 pending entry
保存响应
唤醒对应 waiter 或投递 callback
```

禁止所有同步响应共用一个不校验 Request ID 的 FIFO。

### 12.5 Callback 上下文

RX/Bus Task 不得直接执行可能阻塞的用户 Callback。

推荐：

```text
RX Task
  -> 解析并投递 Event
  -> Application Worker/Executor 执行 Callback
```

Adapter 必须明确 Callback 所在线程和允许的阻塞行为。

---

## 13. Port 接口设计

不要设计一个包含所有平台功能的巨大函数表。按职责拆分：

```text
wlh_transport_ops
wlh_buffer_ops
wlh_osal_ops
wlh_executor_ops
wlh_netif_ops
wlh_power_ops
```

### 13.1 Transport 接口示例

```c
struct wlh_transport_ops {
    int (*start)(void *ctx);
    int (*stop)(void *ctx);
    int (*submit_tx)(void *ctx, struct wlh_buffer *buffer);
    int (*receive)(void *ctx, struct wlh_buffer **buffer, uint32_t timeout_ms);
    int (*reset_peer)(void *ctx);
    size_t (*get_mtu)(void *ctx);
};
```

### 13.2 Buffer 接口示例

```c
struct wlh_buffer_ops {
    struct wlh_buffer *(*alloc)(
        void *ctx,
        size_t payload_size,
        size_t headroom,
        uint32_t flags
    );

    void (*retain)(void *ctx, struct wlh_buffer *buffer);
    void (*release)(void *ctx, struct wlh_buffer *buffer);
    void (*cache_clean)(void *ctx, struct wlh_buffer *buffer);
    void (*cache_invalidate)(void *ctx, struct wlh_buffer *buffer);
};
```

应能适配：

- lwIP `pbuf`；
- Zephyr `net_pkt`；
- NetX Duo packet；
- RT-Thread pbuf；
- 自有固定内存池；
- DMA SRAM；
- 外部 RAM。

### 13.3 OSAL 接口

OSAL 必须为 Host 和 Coprocessor 提供等价的最小语义：

```text
task_create / task_join
mutex_create / lock / unlock
semaphore_create / take / give / give_from_isr
event_create / wait / set / set_from_isr
queue_create / send / receive
timer_create / start / stop
monotonic_time_ms / sleep_ms / yield
in_isr
```

所有等待 API 都必须有有界 Timeout；ISR-safe API 必须显式区分；OSAL 对象的存储、栈和队列容量必须可静态配置。FreeRTOS、Zephyr、POSIX 等实现不得向 Core 泄漏原生句柄类型。

### 13.4 Coprocessor Wi-Fi Adapter 示例

```c
struct wlh_wifi_ops {
    int (*init)(void *ctx);
    int (*deinit)(void *ctx);
    int (*scan_start)(void *ctx, const struct wlh_scan_params *params);
    int (*connect)(void *ctx, const struct wlh_wifi_connect_params *params);
    int (*disconnect)(void *ctx);
    int (*tx_ethernet)(void *ctx, struct wlh_buffer *frame);
    int (*get_capabilities)(void *ctx, struct wlh_wifi_capabilities *caps);
};
```

Wi-Fi RX 通过 Core 回调注入：

```c
wlh_coproc_rx_ethernet(core, interface_id, buffer);
```

---

## 14. Buffer Ownership 与零拷贝

### 14.1 所有权规则必须显式

每个 API 必须明确：

- 调用前谁拥有 buffer；
- 调用成功后所有权是否转移；
- 调用失败后由谁释放；
- Callback 中 buffer 的有效期；
- 是否允许 retain；
- DMA 完成前是否允许修改。

推荐统一约定：

```text
submit_tx 成功：所有权转移给 WL-hosted
submit_tx 失败：所有权仍属于调用者
RX Callback：所有权转移给上层，必须显式 release
```

### 14.2 Headroom

网络栈 buffer 应支持预留 Frame Header Headroom：

```text
[WL-hosted Header][Ethernet Frame]
```

目标是在支持的平台上实现真正零拷贝 TX。

### 14.3 Cache 和 DMA

Adapter 必须处理：

- DMA 地址可达性；
- cache line 对齐；
- clean/invalidate；
- non-cacheable region；
- 外部 RAM 的 DMA 限制；
- scatter-gather 支持。

Core 不得假设所有内存都可直接 DMA。

---

## 15. Flow Control 与调度

### 15.1 按 Channel Credit

不能只维护一个全局“可用 buffer 数”。推荐每个 Channel 独立 Credit：

```text
LINK_CONTROL: 2 个永久保留 Credit
WIFI_RPC:     4
BT_HCI:       4
ETHERNET:    16
OTA:          4
LOG:          2
USER_PASSTHROUGH: 4
```

即使 Ethernet buffer 耗尽，也必须能发送：

- Heartbeat；
- Reset；
- Credit Update；
- Link Health；
- Disconnect；
- Recovery Command。

### 15.2 Buffer Pool 分离

至少区分：

```text
Control Pool
Data Pool
```

推荐进一步区分 RX/TX Pool。禁止 Ethernet 流量耗尽所有控制面内存。

### 15.3 调度策略

推荐：

```text
LINK_CONTROL：严格高优先级，但限制连续 Burst
WIFI_RPC：高优先级
BT_HCI：低延迟权重
ETHERNET：主要带宽
OTA/LOG：低优先级
USER_PASSTHROUGH：中优先级，限制连续 Burst
```

采用 Weighted Round-Robin、Deficit Round-Robin 或“Strict Priority + Burst Budget”。禁止永久严格优先级导致低优先级 Channel 饥饿。

### 15.4 Backpressure

上层必须能够收到明确背压：

```text
WOULD_BLOCK
NO_CREDIT
NO_BUFFER
LINK_NOT_READY
PEER_CONGESTED
```

禁止在资源不足时无日志、无统计地静默丢弃控制消息。

---

## 16. Transport Binding

所有传输对上层呈现统一 Frame 接口，但各自定义独立 Binding 规范。

### 16.1 SDIO

负责：

- Enumeration；
- CMD52/CMD53；
- Function/Register Layout；
- Device Interrupt；
- Block Size；
- Token/Credit Register；
- Streaming Aggregation；
- DMA Alignment；
- 总线 Reset 和重新枚举；
- 1-bit/4-bit 协商。

SDIO 必须优先考虑高吞吐、批量读写和中断合并。

### 16.2 SPI

负责：

- SPI Mode；
- Full/Half Duplex；
- CS；
- Handshake；
- Data Ready；
- 最大 Transaction；
- Padding；
- DMA；
- 设备 Reset。

### 16.3 UART

负责：

- 字节流 Framing；
- Escape；
- Magic Resync；
- CRC；
- Partial Frame；
- 超时；
- 波特率协商；
- 可选 RTS/CTS。

UART 更适合控制、低速网络或早期验证，不作为最高吞吐目标。

### 16.4 USB

初期 USB Binding 采用“POSIX Host + USB Coprocessor Device”模型，不要求 MCU Host 实现 USB Host。负责：

- USB 枚举、Interface/Class/Subclass/Protocol 标识；
- Bulk IN/Bulk OUT Endpoint 和可选 Interrupt IN Endpoint；
- Endpoint MTU、Transfer Size 和多 Frame Aggregation；
- Partial Transfer、Short Packet 和 ZLP 规则；
- Hotplug、Disconnect、Re-enumeration 和 Coprocessor Reset 检测；
- USB Device 唯一标识与多设备选择；
- Host 取消、Timeout 和异步 Transfer 生命周期；
- 控制面与数据面的调度和背压。

WL-hosted Frame 不得依赖 USB packet 边界；一次 USB Transfer 可包含一个或多个 Frame，一个 Frame 也可跨多次 Transfer。USB 自身的可靠传输不代替 Session、Credit、Heartbeat 和上层恢复机制。

---

## 17. 链路协商与能力发现

启动时双方交换：

```text
protocol major/minor range
session_id / boot_id
implementation version
chip/vendor ID
firmware build ID
supported services
每个 service 的版本范围
supported channels
maximum frame size
maximum RPC payload
maximum aggregate size
DMA/cache alignment
initial credits
checksum capabilities
transport capabilities
feature bitmap
optional service capabilities
IO pin capabilities
ADC pin capabilities
KV maximum key/value length
KV total capacity and remaining capacity
USER_PASSTHROUGH maximum RPC payload
USER_PASSTHROUGH maximum stream message size
```

规则：

- Major 无共同版本：拒绝建立链路；
- Minor 不一致：选择双方共同支持的最高版本；
- 未知 Optional Service：忽略；
- Mandatory Service 缺失：明确失败；
- 未知 protobuf 字段：跳过；
- Coprocessor Session 改变：立即取消所有旧 pending RPC。

---

## 18. Session、心跳与恢复

### 18.1 Session ID

Coprocessor 每次启动生成新的 `session_id`。Host 检测到 Session 变化后必须：

```text
取消所有 pending RPC
清空旧 Credit
丢弃旧 Session Frame
重置 Sequence
重新能力协商
重建 Netif/Link 状态
通知应用连接失效
```

### 18.2 Sequence

每个方向或每个 Channel 维护 Sequence，用于：

- 检测丢包；
- 检测重复包；
- 调试乱序；
- 统计链路质量。

是否自动重传由具体 Transport Binding 决定，通用 Frame 层不应默认所有总线都不可靠。

### 18.3 Heartbeat

Heartbeat 必须使用保留控制资源，包含：

- Session；
- Link State；
- Queue High-Water Mark；
- Credit 摘要；
- 错误计数；
- Uptime；
- 可选看门狗状态。

### 18.4 Link Health

公共状态建议：

```c
enum wlh_link_health {
    WLH_LINK_DOWN,
    WLH_LINK_NEGOTIATING,
    WLH_LINK_HEALTHY,
    WLH_LINK_CONGESTED,
    WLH_LINK_PEER_UNRESPONSIVE,
    WLH_LINK_RECOVERING,
    WLH_LINK_FAILED,
};
```

至少提供：

- 最后一次 RX 时间；
- 最后一次 Heartbeat；
- TX/RX Queue 占用；
- Buffer Allocation Failure；
- CRC Error；
- Sequence Gap；
- RPC Timeout；
- Peer Reset Count；
- Transport Reset Count。

### 18.5 恢复层级

按从轻到重执行：

```text
1. 重试单次总线事务
2. 重置 Channel/Queue
3. 重新同步 Frame Stream
4. 重新协商 Link Session
5. 软复位 Coprocessor
6. 重新初始化总线
7. Host 应用决定是否整机复位
```

整机复位只能是最终兜底，不应是唯一恢复路径。

---

## 19. 错误模型

错误码必须分域：

```text
WLH_STATUS_DOMAIN_PROTOCOL
WLH_STATUS_DOMAIN_TRANSPORT
WLH_STATUS_DOMAIN_LINK
WLH_STATUS_DOMAIN_WIFI
WLH_STATUS_DOMAIN_BLUETOOTH
WLH_STATUS_DOMAIN_OTA
WLH_STATUS_DOMAIN_PERIPHERAL
WLH_STATUS_DOMAIN_STORAGE
WLH_STATUS_DOMAIN_DEVICE_INFO
WLH_STATUS_DOMAIN_USER
WLH_STATUS_DOMAIN_VENDOR
```

公共错误示例：

```text
OK
INVALID_ARGUMENT
NOT_SUPPORTED
NOT_READY
BUSY
WOULD_BLOCK
NO_MEMORY
NO_CREDIT
TIMEOUT
CANCELLED
VERSION_MISMATCH
SESSION_CHANGED
TRANSPORT_FAILURE
PEER_UNRESPONSIVE
AUTHENTICATION_FAILED
```

厂商错误码可以作为诊断字段返回，但不得直接成为公共 API 的唯一语义。

---

## 20. 版本管理

### 20.1 四类版本必须分离

```text
Wire Protocol Version
Service Version
Core/Adapter Software Version
Coprocessor Firmware Version
```

不得使用某个 Git Tag 代替 Wire Compatibility 判断。

### 20.2 Protocol Version

推荐：

```text
Major：不兼容的 Frame/语义变化
Minor：向后兼容的能力增加
```

### 20.3 Service Version

每个 Service 独立协商版本，避免 Wi-Fi Service 的变化强迫 OTA 或 Bluetooth Service 同时升级。

### 20.4 仓库版本

各仓库独立 SemVer：

```text
wl-hosted-protocol          v1.2.0
wl-hosted-host-core         v0.6.1
wl-hosted-coproc-core       v0.6.3
wl-hosted-host-hpm-sdk      v0.3.0
wl-hosted-coproc-esp-idf    v0.5.2
```

### 20.5 Host Preset Lock

Host Example 应生成锁定文件：

```yaml
preset: hpm6750evk__esp32c6-devkitc__sdio4
coprocessor_profile: espressif.esp32c6.devkitc.sdio4-wifi-ble
coprocessor_version: 0.3.2
manifest_sha256: "..."
protocol_major: 1
```

Lock 用于可复现开发，不代替运行时协商。

---

## 21. 安全与固件可信性

至少支持：

- Release Artifact SHA-256；
- Release Manifest 签名；
- Catalog 公钥固定；
- OTA Image 签名校验；
- 防止错误 Profile 烧入；
- 防止降级到不兼容或已撤销版本；
- 对敏感 RPC 进行权限划分；
- IO 配置、IO 写入和 ADC 读取必须遵循设备的访问控制策略；
- KV 写入/擦除与 User Passthrough 发送必须可按产品安全策略禁用或授权；
- 对 Debug/Vendor Channel 提供生产禁用选项。

协议层安全和 Wi-Fi 链路加密是不同问题。SDIO/SPI 板内链路默认可能不加密，但产品应根据威胁模型决定是否增加认证、完整性保护或会话密钥。

---

## 22. 测试体系

### 22.1 Protocol Test Vector

必须覆盖：

```text
结构 -> 预期 Wire Bytes
Wire Bytes -> 预期结构
未知字段
边界长度
非法长度
错误 CRC
错误 Magic
错误版本
截断 Frame
重复 Frame
旧版本兼容
```

系统能力 Service 必须额外覆盖：

```text
IO 配置、读写、非法 pin、非法 mode、权限错误
ADC 正常读取、非法 pin、不支持 ADC、校准失败
KV 写入、读取、覆盖、擦除、断电恢复、长度超限
Device Information 字段完整性、UID 和 MAC 格式
User Passthrough RPC 异步 response、Timeout、重复和 Session 变化
User Passthrough Raw Stream 分片、聚合、Credit、不完整消息和大消息
```

### 22.2 POSIX Simulator

在没有硬件的情况下，Host 和 Coprocessor 都运行 POSIX OSAL：

```text
Host Core
  ↕ virtual transport / socketpair
Coprocessor Core
  ↕ mock Wi-Fi backend
```

第一个有真实 Coprocessor 的端到端落地方案为：

```text
POSIX Simulator Host (POSIX OSAL + USB Host Adapter)
  ↕ USB Bulk
ESP32-S3 Coprocessor (ESP-IDF OSAL + USB Device Adapter)
  ↕ Espressif Wi-Fi/Bluetooth backend
```

POSIX Host 需同时支持 virtual transport 和真实 USB transport，以便将同一套 Host Core 测试从 mock Coprocessor 平滑迁移到真实硬件。

POSIX mock Coprocessor 必须注册所有支持的标准系统能力 Service，并对每个无法支持的 Service 返回能力缺失或 `NOT_SUPPORTED`。

支持注入：

- 丢包；
- 重复；
- 乱序；
- 截断；
- CRC 错误；
- 延迟；
- Coprocessor Reset；
- Host Reset；
- Buffer OOM；
- Credit 丢失；
- RPC Timeout；
- Session 改变；
- Queue 饥饿。

### 22.3 macOS 图形化模拟器测试方案

在 POSIX Simulator 之上，macOS 平台提供一套带 GUI 的模拟器测试环境，用于可视化调试、人工故障注入和真实 USB Coprocessor 联调。该方案由三个仓库组成：

- `wl-hosted-host-macos-sim`：Host Core + POSIX OSAL 的 macOS 可执行模拟器，通过虚拟 transport 连接到 Sim Manager。
- `wl-hosted-coproc-macos-sim`：Coprocessor Core + POSIX OSAL + mock Wi-Fi/Bluetooth backend 的 macOS 可执行模拟器，通过虚拟 transport 连接到 Sim Manager。
- `wl-hosted-macos-sim-manager`：基于 Slint + Rust 的 GUI 工具。Rust 实现核心转发、协议解析、子进程管理与故障注入逻辑；Slint 负责跨平台 UI 与按钮布局。

#### 22.3.1 组成与角色

`wl-hosted-macos-sim-manager` 负责：

- 启动/停止 host sim 与 coproc sim 进程；
- 在两者之间转发 WL-hosted Frame；
- 可选：将 host sim 对接到真实 USB Coprocessor；
- 解析并展示链路状态、运行时统计和测试专用监控信息；
- 提供按钮注入各类故障。

所有业务数据和控制数据都经过 Sim Manager，Manager 可对每一帧的 channel、session、sequence、service/method、payload 长度和校验状态进行展示与记录。

#### 22.3.2 两种运行模式

1. **双模拟模式**

   ```text
   Host Sim (macOS)  ←→  Sim Manager (Slint + Rust, macOS)  ←→  Coproc Sim (macOS)
   ```

   全部数据经过 Sim Manager；Manager 可展示每一帧的 channel、session、sequence、service/method、长度、校验状态，并解析监控协议获取 host/coproc sim 内部运行时信息。

2. **模拟 Host + 真实 USB Coprocessor 模式**

   ```text
   Host Sim (macOS)  ←→  Sim Manager (Slint + Rust, macOS)  ←→  USB Coprocessor (真实硬件)
   ```

   Sim Manager 作为 USB Host 侧代理，将 Host Sim 的虚拟帧转发到 USB 设备；同时继续展示标准 Heartbeat、Link 状态以及模拟器侧监控信息。此模式用于在真实 Coprocessor 上复用同一套 Host Sim 调试能力。

#### 22.3.3 监控协议扩展

为让 Sim Manager 实时观测 host/coproc sim 内部状态，在模拟器与 Sim Manager 之间额外定义一条 **Sim Monitor Channel**：

- 使用 Experimental Channel 范围，建议分配 `0x40 SIM_MONITOR`。
- 该 Channel 仅在 macOS 模拟器与 Sim Manager 之间使用，真实固件不启用；通过 Hello 能力协商中的 `channels` 声明，未协商时不得发送。
- 承载两类消息：
  - **SimRuntimeInfo Event**：由 Host/Coproc Sim 周期性发送，包含模拟器内部状态，例如 mock Wi-Fi 状态、当前扫描/连接阶段、待处理 RPC 数量、buffer pool 水位、线程状态、虚拟 transport 队列深度等。
  - **SimFaultInject Request/Response**：由 Sim Manager 向指定侧（host 或 coproc）发送，请求注入某种故障；响应返回是否成功注入。

设计约束：

- 监控协议不得影响标准 Frame 的语义、Session/Credit/Sequence 行为。
- Sim Manager 转发标准 Frame 时保持字节原样，仅对 `SIM_MONITOR` Channel 的消息进行解释或修改。
- 若 Sim Manager 未启动监控，Host/Coproc Sim 应能降级为直接对连或静默不发送监控消息。

#### 22.3.4 Sim Manager 故障注入能力

GUI 提供可点击按钮，向 host sim、coproc sim 或转发路径注入：

- 转发层故障：丢帧、重复帧、乱序帧、延迟帧、篡改 checksum、截断 payload。
- 链路层故障：强制 Session 改变、模拟 Coprocessor 重启、模拟 Host 复位、清空 Credit、重置 Channel。
- 资源故障：触发 Host/Coproc Sim 内部 buffer OOM、限制 Credit 供给、制造 Queue 饥饿。
- 业务故障：强制 mock Wi-Fi 断开、扫描失败、RPC 超时、OTA 中断。

所有注入操作通过 `SimFaultInject` 协议下发，并在 GUI 日志区显示时间戳与参数。

#### 22.3.5 与 POSIX Simulator 的关系

- 22.2 的 POSIX Simulator 以命令行/脚本方式运行，强调 CI、回归测试和无头环境。
- 22.3 的 macOS Sim Manager 强调可视化调试、人工故障注入和真实 USB Coprocessor 联调。
- 两者共享同一套 Host Core、Coprocessor Core 和 POSIX OSAL；区别仅在于 transport 端点：POSIX Simulator 使用 socketpair/pipe 直连，macOS 方案通过 Sim Manager 中转。

### 22.4 Core CI

每次提交至少运行：

- 单元测试；
- ASan；
- UBSan；
- Fuzz Smoke Test；
- 多编译器构建；
- 静态分析；
- 生成文件一致性检查。

### 22.5 Adapter CI

Host Adapter：

- 所有官方 Board 编译；
- 所有 Example 编译；
- Preset Schema 校验；
- 依赖版本校验。

Coprocessor Adapter：

- 所有 Stable Profile 构建；
- Factory/OTA Artifact 生成；
- Manifest 校验；
- 固件大小限制；
- 签名和 Hash 校验。

### 22.6 初期真实硬件验证矩阵

| 硬件/用途 | Host | Coprocessor | Host 生态 | Coprocessor SDK | Transport | 初期定位 |
|---|---|---|---|---|---|---|
| ESP32-S3 核心板 + POSIX 主机 | POSIX 模拟器 | ESP32-S3 | POSIX | ESP-IDF | USB | 首个真实 Coprocessor 与 USB Binding 验证 |
| 慧勤智远 ESP32-P4 + C6 开发板 | ESP32-P4 | ESP32-C6 | ESP-IDF | ESP-IDF | SDIO | 同厂商双 SoC 高吞吐验证 |
| AliOS Things Developer Kit | STM32L496 | BK7231 | STM32Cube HAL + zephyr | bk72xx_freertos_sdk | UART | 跨厂商、低速链路与重同步验证 |
| 自绘开发板 | HPM5301 | BL616 | HPM SDK + RT-Thread | bouffalo_sdk | SPI + 2IO | 跨厂商 SPI 高速数据与双侧带 IO 握手验证 |

`SPI + 2IO` 的两根侧带信号的方向、有效电平和时序必须在 SPI Transport Binding 和 Board 定义中明确，不写死在 Core。

### 22.7 HIL

Tier-1 组合每次 Release 运行：

- Link Establish；
- Scan；
- Connect/Disconnect；
- DHCP；
- Ping；
- TCP/UDP iperf；
- Coprocessor Reset Recovery；
- Host Reset Recovery；
- 长时间压力测试；
- OOM/背压测试；
- IO/ADC/KV/Device Information/User Passthrough 能力矩阵测试；
- Coprocessor OTA；
- 固件版本不兼容测试。

### 22.8 支持等级

| 等级 | 要求 |
|---|---|
| Tier 1 | 每个 Release 构建并运行真实硬件 HIL |
| Tier 2 | 每个 Release 编译，周期性硬件验证 |
| Tier 3 | 社区维护，尽力支持 |
| Experimental | API、性能或稳定性可能变化 |

只为实际验证过的组合建立 Preset。理论可组合不等于官方支持。

---

## 23. Release 流程

### 23.1 Coprocessor Release

```text
更新 Coprocessor Adapter
        ↓
构建所有 Profile
        ↓
运行单元测试和 HIL
        ↓
生成 Factory/OTA/Manifest/Hash/Signature
        ↓
发布 GitHub Release
        ↓
更新 Firmware Catalog
        ↓
Host Preset CI 验证推荐版本
```

### 23.2 Host Adapter Release

```text
更新 Host Adapter/Core 依赖
        ↓
构建所有 Board × Example
        ↓
校验所有 Preset 和 Catalog 引用
        ↓
运行 Tier-1 HIL
        ↓
发布 Host Adapter
```

### 23.3 撤销固件

Catalog 必须支持：

```text
revoked: true
reason: "..."
replacement_version: "..."
```

工具默认拒绝下载已撤销版本，除非用户显式覆盖。

---

## 24. 推荐开发路线

### Phase 0：规范

完成：

- 名称和仓库边界；
- Frame Header；
- Channel；
- RPC Envelope；
- Session；
- Credit；
- 版本规则；
- Error Model；
- 最小 Wi-Fi Service。

### Phase 1：POSIX 原型

完成：

- Host Core；
- Coprocessor Core；
- POSIX OSAL；
- virtual transport；
- mock Wi-Fi backend；
- nanopb 编码；
- 错误注入测试。

### Phase 2：POSIX Host + USB Coprocessor

只实现：

- POSIX Host Adapter 和 POSIX OSAL；
- ESP32-S3 ESP-IDF Coprocessor Adapter 和 FreeRTOS OSAL；
- USB Host/Device Transport Binding；
- Wi-Fi STA；
- Scan；
- Connect/Disconnect；
- Ethernet TX/RX；
- Heartbeat；
- Reset Recovery；
- iperf。

该阶段用 POSIX virtual transport 保留可重复故障注入，再将同一 Host Core 切换到 USB 连接的 ESP32-S3 真实 Coprocessor。

### Phase 3：真实 MCU 组合扩展

按初期硬件矩阵逐步实现：

1. ESP32-P4 + ESP32-C6 / ESP-IDF + ESP-IDF / SDIO；
2. STM32L496 + BK7231 / STM32Cube HAL + bk72xx_freertos_sdk / UART；
3. HPM5301 + GD32VW552 / HPM SDK + GD32VW55x_WiFi_BLE_SDK / SPI + 2IO。

每引入一种新 RTOS/SDK，都必须先完成 OSAL 一致性测试，再进入 Transport 和 Wi-Fi 联调。

### Phase 4：第二个 Coprocessor 厂商

验证：

- 标准 Wi-Fi Service 是否真正厂商无关；
- Vendor Extension 是否足够；
- Firmware Profile/Catalog 模型是否可扩展。

### Phase 5：功能扩展

按优先级增加：

- SoftAP；
- Bluetooth HCI；
- Coprocessor OTA；
- 低功耗；
- 诊断和日志；
- Enterprise Wi-Fi；
- 高级 Vendor Service。

### Protocol 1.0 条件

至少满足：

- 两个 Host 生态；
- 两个 Coprocessor 厂商或 SDK；
- USB、SDIO、UART 和 SPI 四种 Transport Binding；
- Host 和 Coprocessor 的 OSAL 一致性测试；
- 稳定的恢复机制；
- 长时间压力测试；
- 固件 Catalog 和签名流程；
- 兼容性测试覆盖前一 Minor 版本。

---

## 25. Agent 开发硬性规则

以下规则供自动化 Agent 和人工开发者共同遵循。

### 25.1 架构边界

1. Protocol 仓库不得依赖 RTOS、HAL、网络栈或厂商 SDK。
2. Host Core 不得包含具体 Host Board 引脚。
3. Coprocessor Core 不得调用具体厂商 Wi-Fi API。
4. Host Adapter 不得依赖 Coprocessor Adapter 源码。
5. Coprocessor Adapter 不得依赖 Host Adapter 源码。
6. 完整应用 Example 只放在 Host Adapter。
7. Coprocessor 主要以 Release Firmware Profile 交付。
8. Host 和 Coprocessor 均必须通过 OSAL 运行于 RTOS 语义上，不支持裸机/Poll-only Core。
9. Core 不得直接依赖任何具体 RTOS 或 POSIX API。

### 25.2 Wire Protocol

1. 不得直接发送 C struct。
2. 不得使用 C bitfield 作为 Wire Format。
3. 所有多字节字段必须明确端序。
4. 所有长度必须在使用前验证。
5. 所有外部输入都视为不可信。
6. 已发布 Channel、Service、Method 和 protobuf tag 不得复用。
7. 新字段必须具备向后兼容策略。
8. 大数据不得进入 protobuf RPC。

### 25.3 RPC

1. 响应必须按 `request_id` 精确匹配。
2. 响应必须验证 `session_id`。
3. 用户 Callback 不得在 Bus ISR 或 RX 关键路径直接执行。
4. 每个请求必须有 Deadline 或明确的无限等待策略。
5. Pending Request 数量必须有上限。
6. Session 改变时必须取消所有 Pending Request。

### 25.4 内存与并发

1. Core 默认不得依赖无界 malloc。
2. Buffer 所有权必须写入 API 注释和测试。
3. Control 和 Data 资源必须隔离。
4. 不得让高流量 Channel 饿死 Link Control。
5. 不得在 ISR 中进行 protobuf 解码或复杂分配。
6. 不得在持有总线锁时调用用户 Callback。
7. 所有 Queue 和 Pool 必须提供 High-Water Mark 统计。

### 25.5 API 与厂商隔离

1. 公共 API 不得暴露厂商内部结构。
2. 厂商特性必须放入显式 Vendor Service。
3. Vendor Service 仍需版本化和边界检查。
4. 标准 Service 只表达跨厂商可实现的语义。

### 25.6 Example 与固件

1. Example 必须只依赖 Host 公共 API，不直接调用 Coprocessor 厂商 API。
2. Example、Board 和 Preset 必须分离。
3. Preset 必须引用 Firmware Profile，而不是 Coprocessor 源码路径。
4. 固件下载必须校验 Hash；正式通道必须校验签名。
5. 构建结果必须记录精确依赖版本和 Profile Manifest。

### 25.7 修改流程

涉及以下内容的修改必须新增 ADR：

- Frame Header；
- Channel 编号；
- RPC Envelope；
- Error Model；
- Version Negotiation；
- Buffer Ownership；
- Credit Model；
- 仓库边界；
- Firmware Profile Schema。

Agent 在实现功能前应先确认：

```text
此修改属于 Protocol、Core、Adapter、Board、Preset、Example 还是 Firmware Profile？
```

若无法明确归属，先停止编码并补充架构决策。

---

## 26. ADR 建议

在 `wl-hosted-protocol/spec/adr/` 维护：

```text
0001-project-terminology.md
0002-control-data-plane-separation.md
0003-wire-header-format.md
0004-nanopb-as-default-codec.md
0005-host-centric-example-model.md
0006-coproc-firmware-profile.md
0007-per-channel-credit-flow-control.md
0008-session-and-recovery-model.md
0009-repository-boundaries.md
0010-vendor-extension-model.md
0011-standard-system-services.md
0012-user-passthrough-channel.md
```

每个 ADR 包含：

```text
Context
Decision
Alternatives
Consequences
Compatibility Impact
Migration Plan
```

---

## 27. 新 Host Adapter 检查表

- [ ] Core 未被修改为依赖该 SDK
- [ ] 实现 Transport Ops
- [ ] 实现 OSAL 并通过一致性测试
- [ ] 实现 Buffer Ops
- [ ] 实现 Executor/Time Ops
- [ ] 完成 Cache/DMA 规则
- [ ] 完成网络栈接入
- [ ] 至少一个 Board
- [ ] 至少一个 Coprocessor Preset
- [ ] `wifi_station` 可运行
- [ ] `wifi_iperf` 可运行
- [ ] Coprocessor Reset 后可恢复
- [ ] Host Reset 后可恢复
- [ ] OOM/No Credit 行为可观测
- [ ] 支持的 IO/ADC/KV/Device Information/User Passthrough Service 与能力协商一致
- [ ] IO 配置、读写和 ADC 读取可测试
- [ ] KV 原子写入、读取和擦除可测试
- [ ] Device Information 可返回完整结构化字段
- [ ] User Passthrough RPC 与 Raw Stream 可测试
- [ ] 文档说明 Callback 上下文
- [ ] CI 编译所有 Example

---

## 28. 新 Coprocessor Adapter/Profile 检查表

- [ ] Core 未泄漏厂商 SDK 类型
- [ ] 实现 Wi-Fi Ops
- [ ] 实现 Transport Device Ops
- [ ] 实现 OSAL 并通过一致性测试
- [ ] 实现 Ethernet RX/TX
- [ ] 实现 Link Capability
- [ ] 实现 Heartbeat
- [ ] 实现 Host Reset 检测
- [ ] 实现软复位与重新协商
- [ ] 定义 Factory Image
- [ ] 定义 OTA Image
- [ ] 生成 Manifest
- [ ] 生成 SHA-256 和签名
- [ ] 发布到 Firmware Catalog
- [ ] 至少一个 Host Preset 验证
- [ ] 按 Profile 实现可选 IO/ADC/KV/Device Information/User Passthrough Service
- [ ] 报告逻辑 pin、ADC、KV 限制和透传能力
- [ ] 实现授权策略下的 IO 写入、KV 写入/擦除和用户透传
- [ ] 通过所有标准能力 Service 的 POSIX/mock 单元测试
- [ ] 长时间吞吐和恢复测试通过

---

## 29. 默认决策摘要

在没有新的 ADR 推翻前，项目采用以下默认决策：

```text
系统角色：Host / Coprocessor
应用开发中心：Host
Coprocessor 交付：预编译 Release Firmware Profile
Example 位置：Host Adapter
组合配置：Host Preset
协议 schema：.proto
默认 C 实现：nanopb
控制面：RPC
数据面：Raw Frame/Stream
标准可选 Service：IO / ADC / KV / Device Information / User Passthrough
网络栈位置：Host
Wire Header：显式字节编码，禁止 packed struct ABI
RPC 匹配：request_id + session_id
流控：Per-Channel Credit
调度：高优先级控制 + 有界 Burst + 加权公平
内存：静态和有界资源优先
运行模型：Host/Coprocessor 强制 RTOS 语义 + 平台独立 OSAL
传输：SDIO / SPI / UART / USB
恢复：Session、Heartbeat、分层 Reset
版本：Protocol/Service/Core/Firmware 分离
固件发现：Firmware Catalog
正式固件：Hash + Signature
```

---

## 30. 最终架构边界

WL-hosted 的职责可以用一句话概括：

> **Protocol 定义线上传什么，Core 定义状态机如何工作，Adapter 定义在某个平台上如何运行，Host Example 展示应用如何使用，Firmware Profile 定义 Coprocessor 以什么能力和版本交付。**

最终用户看到的应是一套接近成熟 Wi-Fi Host Driver 的体验，而不是两个必须同时维护和编译的 MCU 工程。
