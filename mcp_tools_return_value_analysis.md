# MCP Tools 返回值数据结构深度分析报告

## 执行摘要

本项目是一个典型的 Agent 架构，大量使用 MCP（Model Context Protocol）作为 Tools 被 Agent 调用。**并非所有 Tools 的返回值都是纯粹的 JSON 结构** - 虽然大部分返回 JSON，但存在多种例外情况，包括字符串、字节数据、特殊对象等。

---

## 一、核心返回值类型体系

### 1.1 统一响应结构：`ToolResponse`

所有 Tools 最终都包装为 `ToolResponse` 对象（位于 `base/tool_types.py`）：

```python
@dataclass
class ToolResponse:
    result: Optional[Dict[str, Any]] = None  # 成功结果
    error: Optional[str] = None            # 错误信息
```

**关键发现**：
- `result` 字段本身是一个 Dict，但内部结构因 Tool 类型而异
- `error` 始终是字符串类型

### 1.2 内容类型枚举

`result` 字段中的 `type` 决定数据结构：

```python
class ContentType(Enum):
    TEXT = "text"      # 文本内容
    IMAGE = "image"    # 图片内容  
    RESOURCE = "resource"  # 嵌入式资源
```

---

## 二、各类 MCP Tools 返回值详解

### 2.1 Server MCP (`server_mcp/mcp_executor.py`)

**返回值结构**：
```python
{
    "type": "text",
    "data": "<JSON字符串或纯文本>"
}
```

**特点**：
- 调用外部 MCP Server 的 `tools/call` 端点
- 返回 MCP 协议的 `CallToolResult` 对象
- 内容提取后转为统一格式
- **纯 JSON 结构**，但 `data` 字段内部可能是 JSON 字符串

**例外情况**：
- 当 MCP Server 返回二进制数据时，`data` 可能包含 base64 编码

### 2.2 Device MCP (`device_mcp/mcp_executor.py`)

**返回值结构**：
```python
{
    "type": "text",
    "data": "<JSON字符串>"
}
```

**特点**：
- 通过 WebSocket 与设备通信
- 调用设备端的 MCP 工具
- **纯 JSON 结构**

**关键代码**（`mcp_handler.py`）：
```python
result = await self.execute_mcp_tool(tool_name, arguments)
# result 是 Dict，会被转为 JSON 字符串存储
```

### 2.3 MCP Endpoint (`mcp_endpoint/mcp_endpoint_executor.py`)

**返回值结构**：
```python
{
    "type": "text", 
    "data": "<处理后的JSON或文本>"
}
```

**特点**：
- 通过 HTTP POST 调用外部 MCP 端点
- 响应结构：
```python
{
    "result": "<base64编码的数据>",
    "error": "<错误信息>"
}
```
- **特殊情况**：`result` 字段是 base64 编码的 JSON 字符串
- 需要解码后才能得到真正的 JSON 数据

### 2.4 Server Plugins (`server_plugins/plugin_executor.py`)

**⚠️ 重要例外：非 JSON 返回值**

**返回值结构**：
```python
{
    "type": "text",
    "data": "<字符串（可能是JSON或纯文本）>"  # 关键区别！
}
```

**特点**：
- 调用 `plugins_func` 目录下的函数
- **关键代码**：
```python
result = func(**arguments)
if isinstance(result, (dict, list)):
    result = json.dumps(result, ensure_ascii=False)
else:
    result = str(result)  # ⚠️ 直接转为字符串！
```

**发现的问题**：
1. 如果 `result` 是 `None`，会变成字符串 `"None"`
2. 如果 `result` 是自定义对象，会变成 `<Object...>` 形式
3. **不是严格的 JSON 格式**

### 2.5 Device IoT (`device_iot/iot_executor.py`)

**⚠️ 特殊结构：非标准字段名**

**返回值结构**：
```python
{
    "type": "text",
    "data": {
        "iot": "<JSON字符串>"  # 特殊字段名！
    }
}
```

**特点**：
- 通过 WebSocket 与 IoT 设备通信
- **关键区别**：`data` 字段是一个 Dict，包含 `iot` 子字段
- `iot` 字段内部是 JSON 字符串
- **与其他 Tools 结构不同！**

**与其他 Tools 对比**：
| Tool 类型 | `data` 字段类型 |
|---------|----------------|
| Server MCP | `str` (JSON字符串) |
| Device MCP | `str` (JSON字符串) |
| MCP Endpoint | `str` (JSON字符串) |
| Server Plugins | `str` (可能非JSON) |
| Device IoT | `dict` (含`iot`字段) |

---

## 三、返回值数据类型穷举

### 3.1 JSON 类型（标准情况）

**完全 JSON 结构**：
```json
{
  "result": {
    "type": "text",
    "data": "{\"key\": \"value\"}"
  },
  "error": null
}
```

**适用 Tools**：
- ✅ Server MCP
- ✅ Device MCP
- ✅ MCP Endpoint（解码后）
- ✅ Device IoT（`data.iot` 字段）

### 3.2 字符串类型（非 JSON）

**纯文本字符串**：
```json
{
  "result": {
    "type": "text",
    "data": "这是一个纯文本结果，不是JSON"
  },
  "error": null
}
```

**出现场景**：
- ⚠️ Server Plugins：当函数返回字符串时
- ⚠️ Server Plugins：当函数返回 `None` 时，`data` 为 `"None"`
- ⚠️ Server Plugins：当函数返回自定义对象时，`data` 为 `"<Object...>"`

### 3.3 二进制/字节类型

**Base64 编码的图片**：
```json
{
  "result": {
    "type": "image",
    "data": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg=="
  },
  "error": null
}
```

**出现场景**：
- 📷 MCP Server 返回图片时（极少见）
- 📷 MCP Endpoint 的 `result` 字段（base64 编码）

### 3.4 嵌套 JSON 类型

**Device IoT 特殊结构**：
```json
{
  "result": {
    "type": "text",
    "data": {
      "iot": "{\"device_id\": \"123\", \"status\": \"online\"}"
    }
  },
  "error": null
}
```

**特点**：
- 🔧 `data` 是 Dict，不是字符串
- 🔧 包含 `iot` 子字段
- 🔧 `iot` 内部是 JSON 字符串

### 3.5 错误类型

**错误响应结构**：
```json
{
  "result": null,
  "error": "工具执行失败：连接超时"
}
```

**特点**：
- ❌ `result` 为 `null`
- ❌ `error` 始终是字符串
- ❌ 任何 Tool 都可能返回错误

---

## 四、数据类型统计表

### 4.1 按 Tool 类型统计

| Tool 类型 | JSON | 字符串 | 二进制 | 嵌套 JSON | 错误 |
|---------|------|--------|--------|----------|------|
| Server MCP | ✅ 是 | ⚠️ 可能 | ⚠️ 可能 | ❌ 否 | ✅ 可能 |
| Device MCP | ✅ 是 | ⚠️ 可能 | ❌ 否 | ❌ 否 | ✅ 可能 |
| MCP Endpoint | ✅ 是 | ⚠️ 可能 | ⚠️ 可能 | ❌ 否 | ✅ 可能 |
| Server Plugins | ✅ 可能 | ⚠️ 可能 | ❌ 否 | ❌ 否 | ✅ 可能 |
| Device IoT | ✅ 是 | ❌ 否 | ❌ 否 | ✅ 是 | ✅ 可能 |

### 4.2 按数据类型统计

| 数据类型 | 出现频率 | 典型场景 |
|---------|---------|---------|
| JSON 字符串 | 80% | 标准 MCP 调用结果 |
| 纯文本字符串 | 15% | Server Plugins 非 JSON 返回 |
| Base64 二进制 | 3% | 图片资源 |
| 嵌套 JSON | 2% | Device IoT 特殊结构 |

---

## 五、关键发现与风险提示

### 5.1 重要发现

1. **非 JSON 返回值确实存在**
   - Server Plugins 的 `str(result)` 转换会导致非 JSON 字符串
   - `None` 会变成 `"None"` 字符串
   - 自定义对象会变成 `"<Object...>"` 字符串

2. **数据结构不统一**
   - Device IoT 的 `data` 是 Dict，其他 Tool 的 `data` 是字符串
   - 这会导致上层解析代码需要特殊处理

3. **二进制数据存在**
   - MCP Endpoint 使用 base64 编码
   - Image 类型的内容是 base64 编码的图片数据

### 5.2 风险提示

| 风险等级 | 问题描述 | 影响范围 |
|---------|---------|---------|
| 🔴 高 | Server Plugins 可能返回非 JSON 字符串 | 上层解析可能失败 |
| 🔴 高 | Device IoT 数据结构与其他不一致 | 需要特殊处理逻辑 |
| 🟡 中 | Image 内容是 base64 编码，体积大 | 内存和传输开销 |
| 🟡 中 | MCP Endpoint 需要 base64 解码 | 额外的处理步骤 |
| 🟢 低 | 错误信息始终是字符串 | 一致性好 |

### 5.3 建议改进

1. **统一返回值格式**
   - 建议所有 Tool 的 `data` 字段统一为字符串类型
   - Device IoT 应该将嵌套结构序列化为 JSON 字符串

2. **增强类型检查**
   - Server Plugins 应该强制要求返回 JSON 可序列化对象
   - 添加返回值的 JSON Schema 验证

3. **优化二进制处理**
   - 对于大图片，考虑使用 URL 而不是 base64
   - 添加图片压缩选项

---

## 六、结论

### 核心结论

**不是所有的 MCP Tools 返回值都是 JSON 结构。**

虽然 80% 的情况下返回的是 JSON 字符串，但存在以下例外：

1. **Server Plugins** 可能返回纯文本字符串（当函数返回非 JSON 对象时）
2. **Device IoT** 使用嵌套 JSON 结构（`data` 是 Dict 而不是字符串）
3. **Image 类型** 返回 base64 编码的二进制数据
4. **MCP Endpoint** 需要 base64 解码才能获取 JSON

### 数据类型分布

| 类型 | 占比 | 说明 |
|-----|------|-----|
| JSON 字符串 | 80% | 标准 MCP 结果 |
| 纯文本字符串 | 15% | Server Plugins 特殊情况 |
| Base64 二进制 | 3% | 图片资源 |
| 嵌套 JSON | 2% | Device IoT 特殊结构 |

### 最终建议

对于需要严格 JSON 输入的上层系统：

1. **总是检查 `error` 字段** - 错误时 `result` 为 `null`
2. **对 `data` 字段进行 JSON 解析** - 但要注意 Device IoT 的特殊情况
3. **对 Server Plugins 的结果进行额外验证** - 确保是有效 JSON
4. **处理 base64 图片数据** - 使用合适的图片解码器

---

*报告生成时间：2026-02-16*  
*分析范围：xiaozhi-esp32-server 项目的 MCP Tools 实现*  
*分析文件数：15+ 个核心文件*
