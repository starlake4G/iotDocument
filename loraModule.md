### **智能空调LoRa终端 - 通信协议规范 v2.2**

#### **1. 协议概述**

本协议定义了智能空调LoRa终端与中心网关之间的应用层通信标准。

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

| `action` 字段        | 描述                               |
| -------------------- | ---------------------------------- |
| `"set_ac"`           | 控制空调的运行状态。               |
| `"update_config"`    | 远程更新并保存设备的配置参数。     |
| `"reboot"`           | 远程重启设备。                     |
| `"ping"`             | 探测设备是否在线。                 |
| `"get_device_info"`  | 获取设备的详细信息和当前配置。     |
| `"force_report"`     | 强制设备立即上报一次当前状态。     |
| `"set_log_level"`    | 动态设置设备的串口日志输出级别。   |
| `"factory_reset"`    | **【危险】** 将设备恢复到出厂默认设置。 |

##### **3.1 `action: "set_ac"`**

*   **Payload 结构**:
    ```json
    { "action": "set_ac", "params": { ... } }
    ```
*   **`params` 对象字段及选项**:
    *   `power` (Boolean, 可选):
        *   `true`: 开机
        *   `false`: 关机
    *   `mode` (String, 可选):
        *   `"cool"`: 制冷模式
        *   `"heat"`: 制热模式
        *   `"fan"`: 送风模式
        *   `"dry"`: 除湿模式
        *   `"auto"`: 自动模式
    *   `temp` (Integer, 可选): 目标温度，通常范围 `16` - `30`。
    *   `fan` (String, 可选):
        *   `"low"`: 低风速
        *   `"medium"`: 中风速
        *   `"high"`: 高风速
        *   `"auto"`: 自动风速

##### **3.2 `action: "update_config"`**

*   **Payload 结构**:
    ```json
    { "action": "update_config", "params": { ... } }
    ```
*   **`params` 对象字段及选项**:
    *   `ir_profile` (String, 可选): 红外码库品牌名称。
        *   `"GREE"`
        *   `"MIDEA"`
        *   （可根据固件支持扩展更多品牌）
    *   `report_interval` (Integer, 可选): 状态自动上报周期，单位：秒。
    *   `current_threshold` (Float, 可选): 判断空调开启的电流阈值，单位：安培。
    *   `current_cal` (Float, 可选): 电流传感器校准系数（高级调试用）。

##### **3.3 `action: "reboot"`**

*   **Payload 结构**: `{ "action": "reboot" }` (无`params`)

##### **3.4 `action: "ping"`**

*   **Payload 结构**: `{ "action": "ping" }` (无`params`)

##### **3.5 `action: "get_device_info"`**

*   **Payload 结构**: `{ "action": "get_device_info" }` (无`params`)

##### **3.6 `action: "force_report"`**

*   **Payload 结构**: `{ "action": "force_report" }` (无`params`)

##### **3.7 `action: "set_log_level"`**

*   **Payload 结构**:
    ```json
    { "action": "set_log_level", "params": { "level": "debug" } }
    ```
*   **`params.level` 字段选项**:
    *   `"debug"`: (级别4) 最详细日志
    *   `"info"`: (级别3) 普通信息日志（默认）
    *   `"warn"`: (级别2) 警告信息
    *   `"error"`: (级别1) 仅错误信息
    *   `"none"`: (级别0) 关闭所有日志

##### **3.8 `action: "factory_reset"`**

*   **Payload 结构**: `{ "action": "factory_reset" }` (无`params`)

---

#### **4. 上行数据 (Uplink)**

由终端发送至网关。

##### **4.1. `type: "ack"` - 指令响应**

*   **Payload 结构**:
    ```json
    {
      "ack_id": 12345,
      "status": "ok",
      "error_msg": "Optional error string",
      "device_info": { ... }
    }
    ```
*   **`payload` 对象字段及选项**:
    *   `ack_id` (Integer, **必需**): 响应的下行指令的 `msg_id`。
    *   `status` (String, **必需**):
        *   `"ok"`: 指令成功执行。
        *   `"error"`: 指令执行失败。
    *   `error_msg` (String, 可选): 当`status`为`"error"`时，附带的错误信息。
    *   `device_info` (Object, 可选): **仅在响应`get_device_info`指令时出现**。结构如下：
        *   `firmware_version` (String)
        *   `lora_address` (Integer)
        *   `gateway_address` (Integer)
        *   `ir_profile` (String)
        *   `report_interval` (Integer)
        *   `current_threshold` (Float)
        *   `current_cal` (Float)
        *   `uptime_sec` (Integer): 设备已运行时间（秒）。
        *   `free_heap` (Integer): 剩余内存（字节）。

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
        *   `true`: 运行中
        *   `false`: 已停止
    *   `current` (Float, **必需**): 测得的实时电流值（安培）。
    *   `rssi` (Integer, **必需**): 上次接收到网关信号的RSSI（接收信号强度指示）。
    *   `last_ir` (Object, **必需**): 设备**上次发送**的红外指令所代表的状态。
        *   `p` (Boolean): Power (`true`/`false`)
        *   `m` (String): Mode (`"cool"`, `"heat"`, etc.)
        *   `t` (Integer): Temperature
        *   `f` (String): Fan Speed (`"low"`, `"medium"`, etc.)

---

#### **5. 通信流程示例**

**场景: 网关获取设备信息，然后强制其上报状态**

1.  **[DOWNLINK] 网关发送获取设备信息指令 (CMD)**
    `网关(65534) -> 终端(1)`
    ```json
    { "type":"cmd", "from":65534, "msg_id":1900000100, "payload":{ "action":"get_device_info" } }
    ```

2.  **[UPLINK] 终端响应并附带详细信息 (ACK)**
    `终端(1) -> 网关(65534)`
    ```json
    {
      "type": "ack",
      "from": 1,
      "msg_id": 81000,
      "payload": {
        "ack_id": 1900000100,
        "status": "ok",
        "device_info": {
          "firmware_version": "2.2-enhanced",
          "lora_address": 1,
          "uptime_sec": 7200,
          ...
        }
      }
    }
    ```

3.  **[DOWNLINK] 网关发送强制上报指令 (CMD)**
    `网关(65534) -> 终端(1)`
    ```json
    { "type":"cmd", "from":65534, "msg_id":1900000200, "payload":{ "action":"force_report" } }
    ```

4.  **[UPLINK] 终端响应指令 (ACK)**
    `终端(1) -> 网关(65534)`
    ```json
    { "type":"ack", "from":1, "msg_id":82500, "payload":{ "ack_id":1900000200, "status":"ok" } }
    ```

5.  **[UPLINK] 终端立即发送状态报告 (REPORT)**
    `终端(1) -> 网关(65534)`
    ```json
    {
      "type": "report",
      "from": 1,
      "msg_id": 82510,
      "payload": {
        "power_state": true,
        "current": 1.234,
        ...
      }
    }
    ```