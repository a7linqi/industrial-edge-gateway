# 协议说明

本文档定义两层协议：

1. 现场总线：Modbus RTU（F103 ↔ F407）
2. 网关到 Linux：自定义 TCP 应用层协议（F407 ↔ Linux C 服务）

## 1. 物理层与串口参数

| 项目 | 值 |
|---|---|
| 总线 | RS485，半双工 |
| 波特率 | 115200 |
| 数据位 | 8 |
| 校验 | None |
| 停止位 | 1 |
| 终端电阻 | 总线两端各 120Ω |
| 接线 | A/B 双绞，共地 |
| 从站地址 | 1–247；0 广播；248–255 保留 |

115200 下 3.5 字符间隔约 0.3ms，不能靠 1ms 定时器判帧；从站和主站都应使用 UART IDLE/DMA。

## 2. Modbus RTU

### 2.1 功能码

| 功能码 | 含义 | 数量范围 | 说明 |
|---|---|---|---|
| 0x03 | 读保持寄存器 | 1–125 | 参数、控制、状态 |
| 0x04 | 读输入寄存器 | 1–125 | 测量值，只读 |
| 0x06 | 写单个寄存器 | 1 | 回显地址和值 |
| 0x10 | 写多个寄存器 | 1–123 | 响应回显起始地址和数量 |

网关侧单次读取默认不超过 20 个寄存器，避免帧时间超过 50ms 超时。

### 2.2 地址约定

- 协议地址一律 0-based，不使用 4xxxx 表示法。
- 0x03 与 0x04 是两个独立地址空间，同一数值地址可以同时存在。
- 测量值只通过 0x04 读取；参数/控制只通过 0x03 读写。
- 用错地址空间返回异常码 0x02。
- uint32 统一大端 ABCD（高字在前），起始地址偶数对齐。

### 2.3 帧格式

读请求：

```text
[slave_id][function][start_hi][start_lo][qty_hi][qty_lo][crc_lo][crc_hi]
```

读响应：

```text
[slave_id][function][byte_count][data...][crc_lo][crc_hi]
```

写单寄存器请求：

```text
[slave_id][0x06][addr_hi][addr_lo][value_hi][value_lo][crc_lo][crc_hi]
```

写多寄存器请求：

```text
[slave_id][0x10][start_hi][start_lo][qty_hi][qty_lo][byte_count][data...][crc_lo][crc_hi]
```

ADU 最大 256 字节；固定收发缓冲区各 256 字节。

### 2.4 异常响应

```text
[slave_id][function | 0x80][exception_code][crc_lo][crc_hi]
```

| 异常码 | 含义 |
|---|---|
| 0x01 | 非法功能码 |
| 0x02 | 非法数据地址 |
| 0x03 | 非法数据值 |
| 0x04 | 从站设备故障 |

CRC 错误、地址不匹配、广播读：不响应。广播写：执行，但不响应。

### 2.5 CRC16

Modbus RTU 使用 CRC-16/MODBUS：

- 多项式：0xA001（反射）
- 初值：0xFFFF
- 结果异或：0x0000
- 字节序：低字节在前

测试向量：`"123456789"` → `0x4B37`。

## 3. 应用层协议

### 3.1 固定头

```c
#define GW_MAX_PAYLOAD 1024

#pragma pack(push, 1)
typedef struct {
    uint16_t magic;           // 0xA55A
    uint8_t  version;         // 1
    uint8_t  message_type;
    uint32_t sequence;
    uint32_t gateway_id;
    uint32_t timestamp_ms;    // 网关单调 uptime 毫秒，不依赖 RTC
    uint16_t payload_length;  // <= GW_MAX_PAYLOAD
    uint16_t payload_crc;     // CRC16，覆盖 payload
    uint8_t  flags;           // 预留
    uint8_t  reserved[3];     // 预留，置 0
    uint16_t header_crc;      // CRC16，覆盖本字段之前的 24 字节
} GatewayMessageHeader;
#pragma pack(pop)
```

固定头 26 字节。

| 偏移 | 长度 | 字段 | 说明 |
|---|---|---|---|
| 0 | 2 | magic | 0xA55A，大端 |
| 2 | 1 | version | 1 |
| 3 | 1 | message_type | 见 3.2 |
| 4 | 4 | sequence | 每个连接从 1 开始 |
| 8 | 4 | gateway_id | 网关编号 |
| 12 | 4 | timestamp_ms | 网关单调 uptime 毫秒 |
| 16 | 2 | payload_length | 最大 1024 |
| 18 | 2 | payload_crc | 覆盖 payload |
| 20 | 1 | flags | 预留 |
| 21 | 3 | reserved | 置 0 |
| 24 | 2 | header_crc | 覆盖偏移 0–23 的 24 字节 |

### 3.2 消息类型

| 值 | 名称 | 方向 | 说明 |
|---|---|---|---|
| 0x01 | MSG_HELLO | 网关 → 服务 | 连接建立后上报身份和能力 |
| 0x02 | MSG_HEARTBEAT | 网关 → 服务 | 周期心跳 |
| 0x03 | MSG_DEVICE_DATA | 网关 → 服务 | 批量点位数据 |
| 0x04 | MSG_DEVICE_STATUS | 网关 → 服务 | 从站在线/离线状态 |
| 0x05 | MSG_ALARM | 网关 → 服务 | 告警事件 |
| 0x10 | MSG_COMMAND | 服务 → 网关 | 下发命令 |
| 0x11 | MSG_COMMAND_RESPONSE | 网关 → 服务 | 命令响应 |

### 3.3 字节序与 CRC

- 所有多字节字段网络字节序（大端）。
- `payload_crc` 覆盖 payload。
- `header_crc` 覆盖偏移 0–23 的 24 字节。
- CRC 统一用 CRC-16/CCITT-FALSE（0x1021，初值 0xFFFF）。
- 测试向量：`"123456789"` → `0x29B1`。

### 3.4 时间戳

- `timestamp_ms` 是网关单调 uptime，不依赖 RTC，也不用于跨网关对齐。
- Linux 服务落库时使用收到报文的 wall clock（Unix 毫秒，UTC）。
- 网关重启后 uptime 归零，服务端按新连接处理。

### 3.5 连接生命周期

1. TCP 连接建立。
2. 网关发送 `MSG_HELLO`，携带 gateway_id、固件版本、支持的版本。
3. 服务校验 magic、version、CRC，记录网关上线。
4. 网关周期发送 `MSG_HEARTBEAT`（建议 5s），服务超过 15s 未收到则标记离线。
5. 网关发送 `MSG_DEVICE_DATA`，服务按批量点位解析并入库。
6. 任一方关闭连接，服务清理会话；网关重连后重新 HELLO。

心跳间隔、超时时间在 W5 联调时最终固化。

### 3.6 MSG_DEVICE_DATA 载荷

设计约束（具体字段在 W5 固化）：

- 一条消息包含多个点位，避免每寄存器一条消息造成消息风暴。
- 每个点位包含 point_id、quality、原始值和必要的时间信息。
- 单条 payload 不超过 1024 字节。
- 32 位值一次传输，不允许拆成两条消息。
- 点位定义和 point_id 以 [register-map.md](register-map.md) 为准。

### 3.7 TCP 拆包规则

TCP 是字节流，必须处理半包和粘包：

1. 每个连接维护独立接收缓存。
2. 缓存至少 26 字节后，解析固定头。
3. 校验 magic、version、payload_length 上限、header_crc。
4. 按 payload_length 等待完整 payload。
5. 校验 payload_crc。
6. 处理完一帧后，继续解析缓存中的下一帧。
7. 任一步失败：记录错误、断开连接，等待重连。

### 3.8 错误处理

| 错误 | 处理 |
|---|---|
| magic 错误 | 断开连接并记录 |
| version 不支持 | 断开连接并记录 |
| payload_length > 1024 | 断开连接并记录 |
| header_crc 失败 | 断开连接并记录 |
| payload_crc 失败 | 丢弃该帧，记录并断开 |
| sequence 重复 | 计数并丢弃 |
| 半包 | 保留缓存，等待更多数据 |
| 粘包 | 循环解析，直到缓存不足一帧 |
| 心跳超时 | 标记离线，清理会话 |

### 3.9 安全边界

MVP 使用明文 TCP，仅用于桌面实验和演示，不暴露到公网。后续如需远程部署，应增加 TLS、鉴权和网络隔离。
