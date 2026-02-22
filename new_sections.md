### 7.4 MCP 工具提示词处理

MCP (Model Context Protocol) 工具提示词处理机制负责将 MCP Server 的能力转换为 LLM 可理解的工具描述格式，并注入到系统提示词中。

#### 7.4.1 MCP 工具描述转换流程

MCP 工具描述转换遵循以下流程：

```
MCP Server 元数据
    ↓
工具描述解析 (Tool Descriptor Parsing)
    ↓
提示词格式转换 (Prompt Format Conversion)
    ↓
工具列表注入 (Tool List Injection)
    ↓
{{tools_desc}} 占位符替换
```

**设备 MCP 处理代码示例**：

```python
# core/providers/tools/device_mcp/mcp_client.py

class MCPClient:
    """MCP 客户端，负责与 MCP Server 通信并转换工具描述"""
    
    def __init__(self, server_config: dict):
        self.server_url = server_config.get('url')
        self.tools_cache = {}
        
    async def get_tools_description(self) -> str:
        """获取 MCP Server 的工具描述并转换为提示词格式"""
        tools = await self._fetch_tools()
        return self._format_tools_for_prompt(tools)
    
    def _format_tools_for_prompt(self, tools: list) -> str:
        """将工具描述转换为 LLM 可理解的格式"""
        formatted = []
        for tool in tools:
            tool_desc = f"### {tool['name']}\n"
            tool_desc += f"描述: {tool['description']}\n"
            tool_desc += f"参数: {json.dumps(tool['parameters'], ensure_ascii=False)}\n"
            formatted.append(tool_desc)
        return "\n".join(formatted)
```

#### 7.4.2 设备 MCP 与服务端 MCP 的处理差异

| 特性 | 设备 MCP (Device MCP) | 服务端 MCP (Server MCP) |
|------|----------------------|------------------------|
| **部署位置** | 本地设备或边缘节点 | 云端服务器 |
| **连接方式** | 本地 Socket/HTTP | 远程 HTTP/WebSocket |
| **工具发现** | 动态发现本地能力 | 预配置的服务能力 |
| **延迟特性** | 低延迟（本地执行） | 较高延迟（网络往返） |
| **适用场景** | 本地硬件控制、隐私敏感操作 | 云端计算、第三方服务集成 |

**服务端 MCP 配置示例**：

```yaml
# config.yaml
mcp_servers:
  weather_service:
    type: server_mcp
    url: "https://api.weather.com/mcp"
    api_key: "${WEATHER_API_KEY}"
    timeout: 30
    
  calculator_service:
    type: server_mcp
    url: "https://calc.example.com/mcp"
    retry_policy:
      max_retries: 3
      backoff: exponential
```

#### 7.4.3 MCP 能力注入流程

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   系统启动       │────▶│  MCP Server 发现  │────▶│  工具描述获取   │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                                                        │
                              ┌─────────────────────────┘
                              ▼
                       ┌─────────────────┐
                       │  工具格式转换    │
                       └─────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
 ┌─────────────┐        ┌─────────────┐        ┌─────────────┐
 │  缓存存储   │        │ 提示词注入  │        │  热更新     │
 └─────────────┘        └─────────────┘        └─────────────┘
```

---

### 7.5 IoT 设备提示词机制

IoT 设备提示词机制负责将物理设备的能力转换为 LLM 可理解的工具描述，并动态注入到系统提示词中，使 LLM 能够理解和控制 IoT 设备。

#### 7.5.1 设备描述符解析流程

```
设备发现 (Device Discovery)
    ↓
设备描述符获取 (Descriptor Retrieval)
    ↓
描述符解析 (Descriptor Parsing)
    ↓
能力提取 (Capability Extraction)
    ↓
工具描述生成 (Tool Description Generation)
    ↓
{{tools_desc}} 占位符替换
```

**IotDescriptor 类定义**：

```python
# core/providers/tools/device_iot/iot_descriptor.py

class IotDescriptor:
    """IoT 设备描述符类，负责解析设备描述并生成工具描述"""
    
    def __init__(self, device_config: dict):
        self.device_id = device_config.get('device_id')
        self.device_name = device_config.get('name', 'Unknown Device')
        self.description = device_config.get('description', '')
        self.capabilities = device_config.get('capabilities', [])
        self.protocol = device_config.get('protocol', 'mqtt')
        
    def parse_descriptor(self, raw_descriptor: dict) -> dict:
        """解析原始设备描述符"""
        parsed = {
            'device_info': {
                'id': self.device_id,
                'name': self.device_name,
                'description': self.description
            },
            'capabilities': []
        }
        
        for capability in raw_descriptor.get('capabilities', []):
            parsed['capabilities'].append({
                'name': capability['name'],
                'type': capability['type'],
                'description': capability.get('description', ''),
                'parameters': self._parse_parameters(capability.get('parameters', [])),
                'return_type': capability.get('return_type', 'void')
            })
            
        return parsed
    
    def _parse_parameters(self, params: list) -> dict:
        """解析参数定义"""
        parsed = {}
        for param in params:
            parsed[param['name']] = {
                'type': param['type'],
                'required': param.get('required', False),
                'description': param.get('description', ''),
                'default': param.get('default')
            }
        return parsed
    
    def to_tool_description(self, parsed_descriptor: dict) -> str:
        """将解析后的描述符转换为工具描述文本"""
        tool_desc = f"## 设备: {parsed_descriptor['device_info']['name']}\n"
        tool_desc += f"描述: {parsed_descriptor['device_info']['description']}\n\n"
        
        for cap in parsed_descriptor['capabilities']:
            tool_desc += f"### {cap['name']}\n"
            tool_desc += f"- 类型: {cap['type']}\n"
            tool_desc += f"- 描述: {cap['description']}\n"
            
            if cap['parameters']:
                tool_desc += "- 参数:\n"
                for param_name, param_info in cap['parameters'].items():
                    required = "必填" if param_info['required'] else "可选"
                    default = f", 默认: {param_info['default']}" if param_info['default'] else ""
                    tool_desc += f"  - {param_name} ({param_info['type']}, {required}{default}): {param_info['description']}\n"
            
            tool_desc += f"- 返回值: {cap['return_type']}\n\n"
        
        return tool_desc
```

#### 7.5.2 设备能力到工具描述的转换

```
┌──────────────────┐
│   设备能力定义   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  能力类型识别   │
│  - 读取属性     │
│  - 执行动作     │
│  - 订阅事件     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  工具描述生成   │
│  - 工具名称     │
│  - 参数定义     │
│  - 返回值说明   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  提示词注入     │
│  {{tools_desc}} │
└──────────────────┘
```

**设备能力类型映射表**：

| 设备能力类型 | 工具类型 | 描述 | 示例 |
|------------|---------|------|------|
| `property_read` | 查询工具 | 读取设备属性值 | `get_temperature()` |
| `property_write` | 设置工具 | 修改设备属性值 | `set_brightness(80)` |
| `action_invoke` | 动作工具 | 执行设备动作 | `turn_on()` |
| `event_subscribe` | 订阅工具 | 订阅设备事件 | `subscribe_motion_alert()` |

#### 7.5.3 设备状态在上下文中的包含

设备状态通过动态上下文机制注入到提示词中，使 LLM 能够了解设备的实时状态：

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   设备状态采集   │────▶│   状态格式化    │────▶│  提示词模板注入  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                        │                       │
        ▼                        ▼                       ▼
┌──────────────┐          ┌──────────────┐         ┌──────────────┐
│ 属性值读取  │          │ JSON/文本   │         │ {{context}} │
│ 传感器数据  │          │ 结构化数据  │         │ 占位符替换   │
│ 连接状态    │          │              │         │              │
└──────────────┘          └──────────────┘         └──────────────┘
```

---

### 7.6 Memory 注入机制

Memory 注入机制负责管理和注入对话历史到系统提示词中，使 LLM 能够基于上下文进行连贯的对话。系统支持多种 memory provider，包括 mem0ai 和 mem_local_short。

#### 7.6.1 对话历史获取过程

```
用户查询
    ↓
┌─────────────────┐
│  Memory Provider │
│  选择 (Router)  │
└────────┬────────┘
         │
    ┌────┴────┬────────────┐
    ▼         ▼            ▼
┌───────┐ ┌─────────┐ ┌──────────┐
│mem0ai │ │mem_local│ │ 其他实现  │
│(云)   │ │_short   │ │           │
└───┬───┘ │(本地)   │ └─────┬────┘
    │     └────┬────┘       │
    │          │            │
    └──────────┴────────────┘
               │
               ▼
        ┌─────────────┐
        │  历史记录   │
        │  检索/过滤  │
        └──────┬──────┘
               │
               ▼
        ┌─────────────┐
        │   格式化    │
        │  文本/JSON  │
        └──────┬──────┘
               │
               ▼
        ┌─────────────┐
        │ <memory>    │
        │  标签注入   │
        └─────────────┘
```

#### 7.6.2 历史格式化方式

**mem0ai 格式化方式**：

```python
# core/providers/memory/mem0ai/mem0ai.py

class Mem0AI:
    """mem0ai memory provider，支持云端记忆存储"""
    
    def __init__(self, config: dict):
        self.api_key = config.get('api_key')
        self.user_id = config.get('user_id')
        self.base_url = config.get('base_url', 'https://api.mem0.ai')
        
    async def get_formatted_memory(self, query: str = None, limit: int = 10) -> str:
        """获取格式化的记忆内容"""
        memories = await self._fetch_memories(query, limit)
        return self._format_memories(memories)
    
    def _format_memories(self, memories: list) -> str:
        """将记忆记录格式化为文本"""
        if not memories:
            return ""
        
        formatted = []
        for mem in memories:
            # mem0ai 返回的结构化记忆
            role = mem.get('role', 'user')
            content = mem.get('content', '')
            timestamp = mem.get('created_at', '')
            
            formatted.append(f"[{timestamp}] {role}: {content}")
        
        return "\n".join(formatted)
```

**mem_local_short 格式化方式**：

```python
# core/providers/memory/mem_local_short/mem_local_short.py

class MemLocalShort:
    """本地短期记忆 provider，适用于本地缓存对话历史"""
    
    def __init__(self, config: dict):
        self.max_messages = config.get('max_messages', 20)
        self.storage_path = config.get('storage_path', './memory_cache')
        self.session_cache = {}
        
    def get_formatted_history(self, session_id: str) -> str:
        """获取格式化的对话历史"""
        messages = self._get_cached_messages(session_id)
        return self._format_short_term_memory(messages)
    
    def _format_short_term_memory(self, messages: list) -> str:
        """格式化短期记忆为对话文本"""
        if not messages:
            return ""
        
        formatted = []
        for msg in messages[-self.max_messages:]:  # 只保留最近的 N 条
            role = msg.get('role', '')
            content = msg.get('content', '')
            
            if role == 'user':
                formatted.append(f"用户: {content}")
            elif role == 'assistant':
                formatted.append(f"助手: {content}")
            elif role == 'tool':
                formatted.append(f"工具结果: {content}")
        
        return "\n".join(formatted)
```

#### 7.6.3 不同 Memory Provider 的差异

| 特性 | mem0ai | mem_local_short |
|------|--------|-----------------|
| **存储位置** | 云端 (mem0.ai) | 本地文件系统 |
| **数据持久化** | 长期持久化 | 会话级/短期缓存 |
| **检索能力** | 语义检索、向量化搜索 | 时间序列检索 |
| **上下文长度** | 可配置，支持大量历史 | 有限（默认 20 条） |
| **适用场景** | 跨会话记忆、长期记忆 | 单会话上下文、短期记忆 |
| **隐私性** | 数据上传云端 | 数据本地保留 |
| **依赖** | 需要 API Key | 无外部依赖 |

#### 7.6.4 `<memory>` 标签填充机制

```python
# 记忆注入的核心流程

class MemoryInjector:
    """负责将记忆内容注入到提示词模板中"""
    
    async def inject_memory(self, prompt_template: str, session_id: str, user_query: str = None) -> str:
        """
        将记忆注入到提示词模板
        
        流程:
        1. 从所有启用的 memory provider 获取记忆
        2. 格式化记忆内容
        3. 填充 <memory> 标签
        4. 返回完整提示词
        """
        
        # 1. 获取各类记忆
        long_term_memory = await self._get_long_term_memory(user_query)
        short_term_memory = self._get_short_term_memory(session_id)
        
        # 2. 构建 <memory> 标签内容
        memory_content = self._build_memory_tag(long_term_memory, short_term_memory)
        
        # 3. 替换模板中的占位符
        final_prompt = prompt_template.replace("<memory>", memory_content)
        
        return final_prompt
    
    def _build_memory_tag(self, long_term: str, short_term: str) -> str:
        """构建 <memory> 标签内容"""
        parts = []
        
        if long_term:
            parts.append("<long_term_memory>")
            parts.append(long_term)
            parts.append("</long_term_memory>")
        
        if short_term:
            parts.append("<short_term_memory>")
            parts.append(short_term)
            parts.append("</short_term_memory>")
        
        return "\n".join(parts)
```

**short_term_memory_prompt 说明**：

```python
# 短期记忆的提示词格式示例

short_term_memory_prompt = """
以下是最近的对话历史，请基于这些上下文理解用户的意图：

{short_term_memory}

注意：
1. 以上对话按时间顺序排列，最新的在底部
2. 如果用户的问题与历史对话相关，请保持上下文连贯性
3. 如果用户的意图不明确，可以参考历史对话进行推断
"""
```

---

### 7.7 动态上下文详细说明

动态上下文机制通过 `ContextDataProvider` 类在运行时收集和注入会话相关的动态信息，包括设备信息、会话状态、环境变量等。

#### 7.7.1 `{{dynamic_context}}` 包含的信息

| 信息类别 | 字段名 | 说明 | 示例 |
|---------|--------|------|------|
| **设备信息** | `device_name` | 设备名称 | "小智智能音箱" |
| | `device_id` | 设备唯一标识 | "xiaozhi_001" |
| | `device_type` | 设备类型 | "smart_speaker" |
| **会话信息** | `session_id` | 当前会话ID | "sess_20250216_001" |
| | `user_id` | 用户标识 | "user_12345" |
| | `conversation_turn` | 对话轮次 | 5 |
| **时间信息** | `current_time` | 当前时间 | "2025-02-16 14:30:00" |
| | `timezone` | 时区 | "Asia/Shanghai" |
| **环境信息** | `location` | 位置信息 | "上海市浦东新区" |
| | `language` | 当前语言 | "zh-CN" |

#### 7.7.2 设备信息获取流程

```
┌─────────────────┐
│   设备启动      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  加载设备配置   │
│  (device.yaml)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  初始化设备     │
│  信息提供者   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  注册到         │
│ ContextProvider │
└────────┬────────┘
         │
         ▼
