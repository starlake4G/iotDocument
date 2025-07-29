## **嵌入式DTU项目对接文档**

### 0.修改
##### 250729 2143f594 
#### 新增：公共指令通道 (用于设备广播)
为支持对所有在线设备进行广播操作（如在线状态盘点），新增了一个所有设备都会订阅的公共指令主题。

### 1. 项目概述

本项目是一个多功能物联网DTU（数据传输单元），其核心功能分为两大块：

1.  **MQTT管理与遥测通道**: 设备通过连接到标准MQTT代理（Broker），实现远程监控、配置管理、日志上报和固件升级（OTA）。这是与后台管理平台交互的主要通道。
2.  **TCP透明传输通道**: 设备作为一个TCP客户端，与一个指定的数据服务器建立长连接，并执行一套自定义的握手协议。握手成功后，设备进入“透明传输”模式，作为其本地串口（UART）与远程TCP服务器之间的数据桥梁。

后端系统需要实现 **MQTT Broker** 的客户端逻辑以管理设备，并连接到一个 **特定的TCP服务器** 来接收和发送透传数据。

### 2. 核心功能

*   **设备自注册**: 设备使用其唯一的IMEI作为MQTT客户端ID，连接到MQTT Broker。
*   **状态监控**: 定期通过MQTT上报心跳包，包含网络信号、内存使用、DTU连接状态等信息。
*   **远程配置**: 支持通过MQTT下发JSON格式的配置，动态修改所有可配置参数（如服务器IP、端口、心跳间隔等），配置成功后设备将自动重启以应用。
*   **远程日志**: 可通过MQTT指令开启/关闭日志上报，将设备的运行日志（INFO/ERROR级别）实时推送到MQTT主题。
*   **远程控制**: 支持通过MQTT下发指令，如立即重启设备、查询状态、查询配置等。
*   **固件升级(OTA)**: 支持通过MQTT指令触发，从指定的HTTP服务器下载固件并应用升级。
*   **数据透传**: 与专用的TCP服务器建立连接，进行数据双向透明传输。

### 3. 通信协议与接口详解

#### 3.1. MQTT 管理与遥测通道

这是后端管理平台与设备交互的主要方式。

##### **3.1.1. 连接参数**

设备通过`config.json`文件配置以下参数连接到MQTT Broker。

*   `mqtt_broker_address`: MQTT Broker地址
*   `mqtt_broker_port`: MQTT Broker端口
*   `mqtt_client_id`: MQTT客户端ID，**系统会自动获取并使用设备的IMEI号**。
*   `mqtt_username`: 用户名
*   `mqtt_password`: 密码
*   `mqtt_keepalive_s`: 心跳保持时间（秒）

##### **3.1.2. 主题（Topic）结构**

所有MQTT主题遵循以下结构：
`{project_prefix}/{device_type}/{device_id}/{function}/{sub-function}`

*   `project_prefix`: 项目前缀，固定为 `myiot`
*   `device_type`: 设备类型，固定为 `dtu`
*   `device_id`: 设备唯一标识，即**设备IMEI号**

##### **3.1.3. 上行消息 (设备 -> 后端)**

**1. 心跳状态 (Heartbeat)**
*   **Topic**: `myiot/dtu/{device_id}/status/heartbeat`
*   **触发时机**:
    *   定时上报（周期由`mqtt_heartbeat_interval_s`配置，默认60秒）。
    *   收到`get_status`指令时触发一次详细上报。
*   **QoS**: 0
*   **Payload (JSON)**:
    ```json
    {
        "timestamp": 1678886400, // Unix时间戳 (UTC)
        "imei": "861877077633799", // 设备IMEI
        "dtu_tcp_connected": true, // TCP透传通道是否已连接
        "dtu_state": "PASSTHROUGH", // DTU状态机当前状态 (详见附录A)
        // 以下为可选字段，当收到get_status指令时会包含
        "rssi": "25", // 蜂窝网络信号强度 (0-31, 99为未知)
        "heap_free": 15320 // 剩余内存 (bytes)
    }
    ```

**2. 日志信息 (Log)**
*   **触发时机**: 当`log_to_mqtt_enabled`为`true`时，设备产生的日志会实时上报。
*   **Topic**:
    *   `myiot/dtu/{device_id}/log/info` (普通信息)
    *   `myiot/dtu/{device_id}/log/error` (错误信息)
*   **QoS**: 0
*   **Payload (JSON)**:
    ```json
    {
        "timestamp": 1678886401,
        "level": "INFO", // "INFO" 或 "ERROR"
        "message": "Network (PDP) ready." // 日志内容
    }
    ```

**3. 指令响应 (Command Response)**
*   **Topic**: `myiot/dtu/{device_id}/command/response`
*   **触发时机**: 设备执行完后端下发的指令后，会发布响应。
*   **QoS**: 1
*   **Payload (JSON)**:
    ```json
    {
        "timestamp": 1678886402,
        "command_id": "cmd-12345", // 回应的指令ID
        "success": true, // 指令执行是否成功
        "response_payload": { ... } // 响应内容, 结构取决于具体指令
    }
    ```

##### **3.1.4. 下行消息 (后端 -> 设备)**

后端通过向此Topic发布消息来向设备下发指令。

*   **订阅Topic**: `myiot/dtu/{device_id}/command`
*   **QoS**: 设备以QoS 1订阅
*   **通用Payload格式 (JSON)**:
    ```json
    {
        "action": "action_name",
        "command_id": "cmd-12345", // 后端生成的唯一指令ID，设备会原样返回
        "payload": { ... } // 指令相关的参数
    }
    ```

**支持的 `action` 列表:**

| `action` | 描述 | `payload` 示例 | `response_payload` 示例 |
| :--- | :--- | :--- | :--- |
| **`reboot`** | 命令设备立即重启 | `null` 或 `{}` | `{"status": "Rebooting"}` |
| **`get_config`** | 请求设备返回当前的全量配置信息 | `null` 或 `{}` | `{"dtu_remote_server_ip": "61.146.213.147", ...}` (完整的config.json内容) |
| **`set_config`** | 设置设备的新配置。设备会保存配置并重启生效 | `{"dtu_reconnect_delay_ms": 10000, "mqtt_heartbeat_interval_s": 30}` | `{"status": "Saved and rebooting"}` |
| **`get_status`** | 请求设备立即上报一次详细的心跳状态 | `null` 或 `{}` | 无直接响应，设备会向`heartbeat`主题发布一条详细心跳 |
| **`get_act`** | 获取DTU通道的累计收发字节数和文件系统信息 | `null` 或 `{}` | `{"TX": 1024, "RX": 2048, "fs_free_size": 50000}` |
| **`download_ota`**| 命令设备从 **指定的URL** 下载OTA固件包。URL通过`payload`提供。 | `{"url": "http://your-server.com/firmware/new_version.bin"}` | `{"status": "Download started"}` (或失败信息) |
| **`apply_ota`** | 命令设备应用已下载的固件并重启 | `null` 或 `{}` | `{"status": "Applying OTA update and rebooting"}` |
| **`log`** | 动态控制日志输出 | `{"serial": true}` | `{"status": "Success"}` |

---

##### **3.1.4.1. 公共指令通道**

*   **订阅Topic**: `myiot/dtu/public/command`
*   **QoS**: 设备以QoS 1订阅
*   **用途**: 向所有在线设备下发广播指令。
*   **通用Payload格式 (JSON)**:
    ```json
    {
        "action": "action_name",
        "command_id": "optional-broadcast-cmd-id", // 公共指令的ID是可选的
        "payload": { ... }
    }
    ```

**支持的 `action` 列表:**

| `action` | 描述 | `payload` 示例 | 设备行为 |
| :--- | :--- | :--- | :--- |
| **`report_status`** | 请求所有在线设备立即上报一次详细的心跳状态。主要用于平台快速盘点当前在线的设备列表。 | `null` 或 `{}` | 设备收到指令后，会立即向其各自的 `myiot/dtu/{device_id}/status/heartbeat` 主题发布一条详细的心跳包（与 `get_status` 指令效果相同）。 |


#### 3.2. TCP 透明传输通道

此通道用于设备与业务数据服务器间的原始数据交换。后端需要提供一个TCP服务器并实现下述自定义握手协议。

##### **3.2.1. 连接参数**

设备通过`config.json`文件配置以下参数来连接TCP服务器。

*   `dtu_remote_server_ip`: 业务TCP服务器IP地址
*   `dtu_remote_server_port`: 业务TCP服务器端口
*   `handshake_id_a`, `handshake_id_b`: 参与握手计算的ID字节 (16进制，如`"2E"`代表`0x2E`)


##### **3.2.2. 数据传输**
握手成功后，TCP连接进入 `PASSTHROUGH` (透传) 状态透传模式：

*   **服务器 -> 设备**: 服务器通过TCP连接发送的任何数据，都会被设备原样转发到其本地串口。
*   **设备 -> 服务器**: 设备本地串口收到的任何数据，都会被原样通过TCP连接转发到服务器。

连接断开后，设备会根据`dtu_reconnect_delay_ms`配置的延时自动重连，并重新开始握手流程。

---

### 4. 配置文件 `config.json` 详解

这是设备上存储的所有配置项。后端可通过MQTT的`set_config`指令进行修改。

```json
{
    // DTU 透传通道配置
    "dtu_remote_server_ip": "61.146.213.147", // TCP业务服务器IP
    "dtu_remote_server_port": 3313,           // TCP业务服务器端口
    "dtu_reconnect_delay_ms": 5000,           // TCP断线重连延迟(ms)
    "handshake_id_a": 1,                      // 握手ID_A (十进制，代码中作为uint8_t使用)
    "handshake_id_b": 46,                     // 握手ID_B (十进制，代码中作为uint8_t使用)

    // MQTT 管理通道配置
    "mqtt_broker_address": "starlake.tech",   // MQTT Broker地址
    "mqtt_broker_port": 1883,                 // MQTT Broker端口
    "mqtt_client_id": "861877077633799",      // 客户端ID (设备启动时会被IMEI覆盖)
    "mqtt_username": "user",                  // MQTT用户名
    "mqtt_password": "password1",             // MQTT密码
    "mqtt_keepalive_s": 120,                  // MQTT心跳保持(s)
    "mqtt_use_ssl": false,                    // 是否使用SSL/TLS (当前代码未完全支持)

    // 应用行为配置
    "mqtt_heartbeat_interval_s": 60,          // MQTT心跳上报周期(s)
    "log_to_mqtt_enabled": true,              // 是否开启MQTT日志上报
    "g_dtu_debug_enabled": false              // 是否开启本地串口调试日志输出
}
```

### 附录A: DTU状态机 `dtu_state` 枚举值

| 状态值 | 描述 |
| :--- | :--- |
| `IDLE` | 空闲状态，未连接 |
| `TCP_CONNECTING` | 正在连接TCP服务器 |
| `WAITING_S_PKT1` | 已连接，等待服务器的第一个握手包 |
| `PROCESSING_S_PKT1`| 正在处理 S_PKT1 |
| `WAITING_S_PKT2` | 已发送C_PKT1，等待服务器的第二个握手包 |
| `PASSTHROUGH` | 握手成功，进入数据透传模式 |
| `ERROR` | 发生错误，准备重连 |
| `CLOSING` | 正在关闭Socket |