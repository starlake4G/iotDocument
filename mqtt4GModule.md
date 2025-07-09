## DTU 远程控制接口文档 (MQTT)

### 1. 概述

本设备通过 MQTT 协议提供远程监控和管理能力。所有通信都基于 JSON 格式的 payload。设备连接到 MQTT Broker 后，会订阅特定的主题以接收命令和配置，同时向其他主题发布自身的状态和日志信息。

#### 1.1 MQTT 主题结构

主题结构遵循以下模式：`{项目前缀}/{设备类型}/{设备ID}/{功能路径}`

-   **项目前缀 (Project Prefix)**: `myiot` (固定)
-   **设备类型 (Device Type)**: `dtu` (固定)
-   **设备ID (Device ID)**: 设备的 IMEI 号，例如 `867179070640648`。
-   **功能路径**: 定义了消息的具体用途，例如 `command`, `status/heartbeat` 等。

---

### 2. 设备上行主题 (设备 -> 服务器)

设备会主动向以下主题发布消息，用于状态上报和命令响应。

#### 2.1 状态心跳 (Status Heartbeat)

-   **主题**: `myiot/dtu/{设备ID}/status/heartbeat`
-   **QoS**: 0
-   **描述**: 设备定期（默认60秒）或在被请求时发送心跳包，报告其当前运行状态。
-   **Payload (JSON)**:
    ```json
    {
      "timestamp": 1749728266,
      "imei": "867179070640648",
      "dtu_tcp_connected": true,
      "dtu_state": "TCP->UART Relaying ",
      "rssi": "31",
      "heap_free": 1112360
    }
    ```
-   **字段说明**:
    -   `timestamp` (Number): 上报时的 Unix 时间戳 (UTC)。
    -   `imei` (String): 设备的 IMEI 号。
    -   `dtu_tcp_connected` (Boolean): DTU 的 TCP 透传通道是否已连接到目标服务器。
    -   `dtu_state` (String): DTU 透传模块的当前详细状态。
    -   `rssi` (String, 可选): 信号强度指示。仅在详细心跳中出现。
    -   `heap_free` (Number, 可选): 系统剩余堆内存大小。仅在详细心跳中出现。
    -
    ```
        case DTU_STATE_IDLE:                return "IDLE";
    case DTU_STATE_TCP_CONNECTING:      return "TCP_CONNECTING";
    case DTU_STATE_WAITING_S_PKT1:      return "WAITING_S_PKT1";
    case DTU_STATE_PROCESSING_S_PKT1:   return "PROCESSING_S_PKT1";
    case DTU_STATE_WAITING_S_PKT2:      return "WAITING_S_PKT2";
    case DTU_STATE_PASSTHROUGH:         return "PASSTHROUGH";
    case DTU_STATE_ERROR:               return "ERROR";
    case DTU_STATE_CLOSING:             return "CLOSING";
    default:                            return "UNKNOWN_STATE";
    ```
#### 2.2 命令响应 (Command Response)

-   **主题**: `myiot/dtu/{设备ID}/command/response`
-   **QoS**: 1
-   **描述**: 设备在执行完一个下行命令后，通过此主题回复执行结果。
-   **Payload (JSON)**:
    ```json
    {
      "timestamp": 1678886400,
      "command_id": "cmd-12345",
      "success": true,
      "response_payload": {
        "status": "Rebooting"
      }
    }
    ```
-   **字段说明**:
    -   `timestamp` (Number): 响应时的 Unix 时间戳。
    -   `command_id` (String): 响应对应命令的唯一ID，与下发命令中的 `command_id` 一致。
    -   `success` (Boolean): 命令是否成功执行。`true` 表示成功，`false` 表示失败。
    -   `response_payload` (Object/String): 命令的返回数据。
        -   对于 `get_config` 命令，这里是包含完整配置的 JSON 对象。
        -   对于其他命令，通常是一个包含 `status` 信息的简单 JSON 对象。

#### 2.3 其他上行主题 (可选)

根据您的代码，还存在以下主题，但主要用于调试和事件记录：
-   `myiot/dtu/{设备ID}/status/boot`: 设备启动时上报一次启动信息。
-   `myiot/dtu/{设备ID}/status/event`: 上报关键事件，如 DTU 连接状态变化。
-   `myiot/dtu/{设备ID}/log/info`: 上报普通级别的日志。
-   `myiot/dtu/{设备ID}/log/error`: 上报错误级别的日志。

---

### 3. 设备下行主题 (服务器 -> 设备)

服务器通过向以下主题发布消息来控制设备。

#### 3.1 命令下发 (Command Set)

-   **主题**: `myiot/dtu/{设备ID}/command`
-   **QoS**: 1 (建议)
-   **描述**: 向设备下发一个需要立即执行的命令。
-   **Payload (JSON)**:
    ```json
    {
      "command_id": "cmd-12345",
      "action": "reboot",
      "payload": {}
    }
    ```
-   **字段说明**:
    -   `command_id` (String): 命令的唯一标识符。设备会在响应中原样返回此 ID，用于请求和响应的匹配。
    -   `action` (String): 需要执行的具体操作。见下表。
    -   `payload` (Object): 执行命令所需的附加参数。

#### 3.2 支持的 `action` 列表

| `action` 值 | `payload` 内容 | 描述 | 响应示例 (`response_payload`) |
| :--- | :--- | :--- | :--- |
| `reboot` | (空) | 命令设备立即重启。 | `{"status": "Rebooting"}` |
| `get_status` | (空) | 请求设备立即上报一次详细的心跳包到 `status/heartbeat` 主题。 | `{"status": "Success"}` (表示命令已收到) |
| `get_config` | (空) | 请求设备返回其当前正在运行的所有配置参数。 | 包含完整配置的 JSON 对象 (见 3.3 节) |
| `set_config` | 包含要修改的配置项的 JSON 对象 | **更新设备的配置**。设备收到后会应用新配置、保存到闪存并自动重启以使所有配置生效。 | `{"status": "Success"}` |
| `get_act` | (空) | 获取设备自启动以来的网络流量统计和文件系统信息。 | `{"TX": 1024, "RX": 2048, "fs_free_size": 430080}` |
| `log` | `{"serial": true}` 或 `{"serial": false}` | 动态开启或关闭设备的串口调试日志输出。 | (无特定响应) |
| `download_ota` | (空) | (开发中) 命令设备从预设的 URL 下载固件更新包。 | (无特定响应) |
| `apply_ota` | (空) | (开发中) 命令设备应用已下载的固件更新包并重启。 | (无特定响应) |

#### 3.3 **重点: 如何下发配置 (`set_config`)**

这是最重要的远程管理功能之一。通过 `set_config` 命令，您可以修改设备的所有可配置参数。

-   **下发命令示例**:
    假设您想修改 DTU 透传的目标服务器地址和端口，并缩短 MQTT 心跳间隔。

-   **MQTT 主题**: `myiot/dtu/867179070640648/command`

-   **MQTT Payload**:
    ```json
    {
      "command_id": "config-update-001",
      "action": "set_config",
      "payload": {
        "dtu_remote_server_ip": "118.31.16.185",
        "dtu_remote_server_port": 9501,
        "mqtt_heartbeat_interval_s": 30,
        "handshake_id_a": "A1",
        "handshake_id_b": "B2"
      }
    }
    ```

-   **执行流程**:
    1.  设备收到此消息，解析 `payload` 中的 JSON 对象。
    2.  设备将 `payload` 中的值与当前运行的配置进行对比。
    3.  设备将新的值更新到内存中的配置结构体。
    4.  设备将更新后的完整配置保存到设备的非易失性存储（闪存）中。
    5.  设备向 `command/response` 主题回复成功消息。
    6.  **设备自动重启**，以确保所有模块（网络、MQTT、DTU）都加载了最新的配置。

-   **可配置参数列表** (可在 `set_config` 的 `payload` 中使用的所有键):

    | Key (键) | 类型 | 描述 |
    | :--- | :--- | :--- |
    | `dtu_remote_server_ip` | String | DTU TCP 透传的目标服务器 IP 地址。 |
    | `dtu_remote_server_port` | Number | DTU TCP 透传的目标服务器端口。 |
    | `dtu_reconnect_delay_ms`| Number | DTU TCP 连接断开后的重连延迟（毫秒）。 |
    | `handshake_id_a` | String (Hex) | DTU 握手协议的 ID 字节 A，**必须为两位十六进制字符串**，如 "45"。 |
    | `handshake_id_b` | String (Hex) | DTU 握手协议的 ID 字节 B，**必须为两位十六进制字符串**，如 "A1"。 |
    | `mqtt_broker_address` | String | MQTT Broker 的地址。 |
    | `mqtt_broker_port` | Number | MQTT Broker 的端口。 |
    | `mqtt_username` | String | MQTT 连接用户名。 |
    | `mqtt_password` | String | MQTT 连接密码。 |
    | `mqtt_keepalive_s` | Number | MQTT 保持连接的 Keep-Alive 时间（秒）。 |
    | `mqtt_heartbeat_interval_s`| Number | MQTT 心跳包的发送间隔（秒）。 |
    | `log_to_mqtt_enabled` | Boolean | 是否将设备日志通过 MQTT 上报。 |

    **注意**: `mqtt_client_id` 和 `g_dtu_debug_enabled` 等参数是设备内部使用或只读的，不支持远程修改。

---
### 4. 附录：`get_config` 响应示例

当您发送 `{"action": "get_config"}` 命令后，设备会在 `command/response` 主题的 `response_payload` 字段中返回类似以下的完整配置信息：

```json
{
  "dtu_remote_server_ip": "61.146.213.147",
  "dtu_remote_server_port": 3313,
  "dtu_reconnect_delay_ms": 5000,
  "handshake_id_a": "01",
  "handshake_id_b": "45",
  "mqtt_broker_address": "starlake.tech",
  "mqtt_broker_port": 1883,
  "mqtt_client_id": "867179070640648",
  "mqtt_username": "user",
  "mqtt_password": "password1",
  "mqtt_keepalive_s": 120,
  "mqtt_use_ssl": false,
  "mqtt_heartbeat_interval_s": 60,
  "log_to_mqtt_enabled": false,
  "g_dtu_debug_enabled": true
}
```