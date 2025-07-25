### **智能空调LoRa终端 - 通信协议规范 v2.3**

#### **1. 协议概述**

本协议定义了智能空调LoRa终端（固件版本 `v2.3-fs` 及以上）与中心网关之间的应用层通信标准。它支持标准的空调控制以及用户自定义红外指令的动态管理。

*   **传输格式**: 所有应用层数据包均为 **UTF-8编码** 的 **JSON字符串**。
*   **通信模式**:
    *   **下行 (Downlink)**: 网关向终端发送指令 (`type: "cmd"`)。
    *   **上行 (Uplink)**: 终端向网关发送响应 (`type: "ack"`) 或主动上报状态 (`type: "report"`)。
*   **寻址**: 物理层由LoRa模块处理，应用层消息中也包含地址信息用于确认和记录。

---

#### **2. 通用消息结构**

所有JSON消息均遵循以下顶层结构：

| 字段      | 类型    | 描述                                                           |
| --------- | ------- | -------------------------------------------------------------- |
| `type`    | String  | **必需**. 消息的类型。                                         |
| `from`    | Integer | **必需**. 消息发送方的LoRa地址（十进制表示）。                 |
| `msg_id`  | Integer | **必需**. 消息的唯一标识符，建议使用Unix时间戳或`millis()`。 |
| `payload` | Object  | **必需**. 消息的核心数据负载。                                 |

**`type` 参数选项:**
*   `"cmd"`: 指令消息，由网关发出。
*   `"ack"`: 响应消息，由终端发出，用于回应`cmd`。
*   `"report"`: 报告消息，由终端主动发出。

---

#### **3. 下行指令 (Downlink: `type: "cmd"`)**

`payload`中必须包含一个`action`字段来指定具体操作。

| `action` 字段        | 描述                                                       |
| -------------------- | ---------------------------------------------------------- |
| **标准控制与配置**   |                                                            |
| `"set_ac"`           | 控制空调的运行状态 (仅对内置红外Profile有效)。             |
| `"update_config"`    | 远程更新并保存设备的基础配置参数。                         |
| `"reboot"`           | 远程重启设备。                                             |
| `"ping"`             | 探测设备是否在线。                                         |
| `"get_device_info"`  | 获取设备的详细信息、配置和文件系统状态。                   |
| `"force_report"`     | 强制设备立即上报一次当前状态。                             |
| `"set_log_level"`    | 动态设置设备的串口日志输出级别。                           |
| `"factory_reset"`    | **【危险】** 清除所有配置和自定义红外码，恢复出厂设置。      |
| **自定义红外码管理** |                                                            |
| `"send_custom_ir"`   | 发送一个已存储的自定义红外指令。                           |
| `"add_custom_ir"`    | 添加一个新的或覆盖一个已存在的自定义红外指令。             |
| `"delete_custom_ir"` | 从设备的文件系统中删除一个自定义红外指令。                 |
| `"list_custom_ir"`   | 获取设备上存储的所有自定义红外指令的名称列表。             |

##### **3.1 `action: "set_ac"`**

*   **描述**: 适用于 `ir_profile` 设置为内置品牌 (如 "GREE") 的情况。
*   **Payload 结构**:
    ```json
    { "action": "set_ac", "params": { ... } }
    ```
*   **`params` 对象字段及选项**:
    *   `power` (Boolean, 可选): `true` (开) / `false` (关)。
    *   `mode` (String, 可选): `"cool"`, `"heat"`, `"fan"`, `"dry"`, `"auto"`。
    *   `temp` (Integer, 可选): 目标温度，通常 `16` - `30`。
    *   `fan` (String, 可选): `"low"`, `"medium"`, `"high"`, `"auto"`。

##### **3.2 `action: "update_config"`**

*   **Payload 结构**:
    ```json
    { "action": "update_config", "params": { ... } }
    ```
*   **`params` 对象字段及选项**:
    *   `ir_profile` (String, 可选): 红外码库品牌名称或设为`"CUSTOM"`以表明主要使用自定义码。
    *   `report_interval` (Integer, 可选): 状态自动上报周期，单位：秒。
    *   `current_threshold` (Float, 可选): 判断空调开启的电流阈值，单位：安培。
    *   `current_cal` (Float, 可选): 电流传感器校准系数（高级调试用）。

##### **3.3 `action: "send_custom_ir"`**

*   **描述**: 触发一个已存储的自定义红外指令的发送。
*   **Payload 结构**:
    ```json
    { "action": "send_custom_ir", "params": { "name": "power_on" } }
    ```
*   **`params.name`** (String, **必需**): 要发送的指令名称，必须与`add_custom_ir`时使用的`name`完全匹配。

##### **3.4 `action: "add_custom_ir"`**

*   **描述**: 将一个完整的红外指令（包括频率和原始时序数据）保存到设备的文件系统中。如果同名指令已存在，则会**覆盖**。
*   **Payload 结构**:
    ```json
    {
      "action": "add_custom_ir",
      "params": {
        "name": "power_on",
        "freq": 38,
        "data": [9000, 4500, 560, ...]
      }
    }
    ```
*   **`params` 对象字段**:
    *   `name` (String, **必需**): 指令的唯一名称（用作文件名）。
    *   `freq` (Integer, **必需**): 红外载波频率，单位 kHz (e.g., `38`)。
    *   `data` (Array of Integers, **必需**): 原始红外时序数据 (raw timings)。

##### **3.5 `action: "delete_custom_ir"`**

*   **Payload 结构**:
    ```json
    { "action": "delete_custom_ir", "params": { "name": "power_on" } }
    ```
*   **`params.name`** (String, **必需**): 要删除的指令名称。

##### **3.6 `action: "list_custom_ir"`**

*   **Payload 结构**: `{ "action": "list_custom_ir" }` (无`params`)

---

#### **4. 上行数据 (Uplink)**

##### **4.1. `type: "ack"` - 指令响应**

*   **Payload 结构**:
    ```json
    {
      "ack_id": 12345,
      "status": "ok",
      "error_msg": "Optional error string",
      "device_info": { ... },
      "ir_commands": [ ... ]
    }
    ```
*   **`payload` 对象字段及选项**:
    *   `ack_id` (Integer, **必需**): 响应的下行指令的 `msg_id`。
    *   `status` (String, **必需**): `"ok"` (成功) / `"error"` (失败)。
    *   `error_msg` (String, 可选): 当`status`为`"error"`时，附带的错误信息。
    *   `device_info` (Object, 可选): **仅在响应`get_device_info`指令时出现**。
        *   `firmware_version` (String)
        *   `lora_address` (Integer)
        *   `gateway_address` (Integer)
        *   `ir_profile` (String)
        *   `report_interval` (Integer)
        *   `current_threshold` (Float)
        *   `current_cal` (Float)
        *   `uptime_sec` (Integer): 设备已运行时间（秒）。
        *   `free_heap` (Integer): 剩余内存（字节）。
        *   `fs_total` (Integer): 文件系统总空间（字节）。
        *   `fs_used` (Integer): 文件系统已用空间（字节）。
    *   `ir_commands` (Array of Strings, 可选): **仅在响应`list_custom_ir`指令时出现**。包含所有已存储的自定义指令名称。

##### **4.2. `type: "report"` - 状态报告**

*   **Payload 结构**:
    ```json
    {
      "power_state": true,
      "current": 2.513,
      "rssi": -85,
      "last_ir": { ... }
    }
    ```
*   **`payload` 对象字段及选项**:
    *   `power_state` (Boolean, **必需**): 根据电流传感器判断的空调真实运行状态。
    *   `current` (Float, **必需**): 测得的实时电流值（安培）。
    *   `rssi` (Integer, **必需**): 上次接收到网关信号的RSSI。
    *   `last_ir` (Object, **必需**): 设备**上次发送**的红外指令所代表的状态。
        *   **注意**: 对于`send_custom_ir`发送的指令，此对象可能为空`{}`或不出现，因为它不具备标准模式的语义。

---

#### **5. 通信流程示例：自定义红外码管理**

**场景: 网关添加一个新的“关机”指令，列出所有指令，然后触发它。**

1.  **[DOWNLINK] 网关发送添加“关机”指令 (CMD)**
    `网关(65534) -> 终端(1)`
    ```json
    { "type":"cmd", "from":65534, "msg_id":1900001000, "payload":{ "action":"add_custom_ir", "params":{"name":"power_off", "freq":38, "data":[9000, 4500, ...]} } }
    ```

2.  **[UPLINK] 终端响应，确认保存成功 (ACK)**
    `终端(1) -> 网关(65534)`
    ```json
    { "type":"ack", "from":1, "msg_id":91000, "payload":{ "ack_id":1900001000, "status":"ok" } }
    ```

3.  **[DOWNLINK] 网关请求列出所有可用指令 (CMD)**
    `网关(65534) -> 终端(1)`
    ```json
    { "type":"cmd", "from":65534, "msg_id":1900001100, "payload":{ "action":"list_custom_ir" } }
    ```

4.  **[UPLINK] 终端响应并附带指令列表 (ACK)**
    `终端(1) -> 网关(65534)`
    ```json
    {
      "type": "ack", "from": 1, "msg_id": 92500,
      "payload": {
        "ack_id": 1900001100,
        "status": "ok",
        "ir_commands": [ "power_on", "power_off" ]
      }
    }
    ```

5.  **[DOWNLINK] 网关发送“关机”指令 (CMD)**
    `网关(65534) -> 终端(1)`
    ```json
    { "type":"cmd", "from":65534, "msg_id":1900001200, "payload":{ "action":"send_custom_ir", "params":{ "name":"power_off" } } }
    ```

6.  **[UPLINK] 终端响应，确认指令已发送 (ACK)**
    `终端(1) -> 网关(65534)`
    ```json
    { "type":"ack", "from":1, "msg_id":93000, "payload":{ "ack_id":1900001200, "status":"ok" } }
    ```