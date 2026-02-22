# 小智ESP32服务器提示词分析报告

> 生成时间：2026-02-15  
> 分析范围：main/xiaozhi-server 目录下所有提示词相关内容

---

## 目录

1. [概述](#概述)
2. [核心提示词文件](#核心提示词文件)
   - [agent-base-prompt.txt](#agent-base-prompttxt)
   - [system_prompt.py](#system_promptpy)
   - [prompt_manager.py](#prompt_managerpy)
3. [角色提示词](#角色提示词)
4. [意图识别提示词](#意图识别提示词)
5. [工具调用提示词](#工具调用提示词)
6. [提示词架构设计](#提示词架构设计)
7. [总结](#总结)

---

## 概述

小智ESP32服务器是一个智能语音助手项目，采用了**多层级、模块化**的提示词设计架构。整个系统的提示词可以分为以下几个层次：

| 层级 | 说明 | 作用 |
|------|------|------|
| **基础角色层** | agent-base-prompt.txt | 定义AI助手的基础人格和沟通风格 |
| **系统工具层** | system_prompt.py | 定义工具调用的格式和规则 |
| **管理增强层** | prompt_manager.py | 动态注入上下文信息 |
| **角色切换层** | change_role.py | 支持多角色人格切换 |
| **意图识别层** | intent_llm.py | 识别用户意图并路由到对应功能 |

---

## 核心提示词文件

### agent-base-prompt.txt

**文件位置：** `main/xiaozhi-server/agent-base-prompt.txt`

**文件作用：** 这是整个系统最基础的提示词模板，定义了AI助手的基础人格、沟通风格和行为准则。所有其他提示词都以此为基础进行扩展。

**提示词结构分析：**

```xml
<identity>
{{base_prompt}}
</identity>

<emotion>
【核心目标】你不是冰冷的机器...
</emotion>

<communication_style>
【核心目标】使用自然、温暖、口语化的人类对话方式...
</communication_style>

<communication_length_constraint>
【核心目标】所有需要输出长文本内容...单次回复长度不得超过300字...
</communication_length_constraint>

<speaker_recognition>
- 识别前缀：当用户格式为 {...}
</speaker_recognition>

<tool_calling>
【核心原则】优先利用`<context>`信息，仅在必要时调用工具...
</tool_calling>

<context>
【重要！以下信息已实时提供，无需调用工具查询，请直接使用：】
- **当前时间：** {{current_time}}
- **今天日期：** {{today_date}} ({{today_weekday}})
- **今天农历：** {{lunar_date}}
- **用户所在城市：** {{local_address}}
- **当地未来7天天气：** {{weather_info}}
{{ dynamic_context }}
</context>

<memory>
</memory>
```

**各模块作用详解：**

| 模块 | 作用 |
|------|------|
| `<identity>` | 占位符，用于注入用户自定义的基础提示词 |
| `<emotion>` | 定义情感表达方式，包括笑声、惊讶、安慰等情感要素 |
| `<communication_style>` | 定义沟通风格，要求自然、温暖、口语化 |
| `<communication_length_constraint>` | 限制回复长度，单次不超过300字，支持分段讲述 |
| `<speaker_recognition>` | 说话人识别，支持多说话人场景 |
| `<tool_calling>` | 工具调用规则，定义何时调用工具、如何调用 |
| `<context>` | 动态上下文信息，包括时间、天气、位置等 |
| `<memory>` | 记忆占位符，用于注入历史对话记忆 |

---

### system_prompt.py

**文件位置：** `main/xiaozhi-server/core/providers/llm/system_prompt.py`

**文件作用：** 定义工具调用（Function Calling）的系统提示词，用于指导大模型如何理解和使用工具。

**核心函数：**

```python
def get_system_prompt_for_function(functions: str) -> str:
```

**提示词内容：**

```
====

TOOL USE

You have access to a set of tools that are executed upon the user's approval...

# Tool Use Formatting

Tool use is formatted using JSON-style tags...

<tool_call>
{{
    "name": "function name",
    "arguments": {{
        "param1": "value1",
        "param2": "value2",
    }}
}}
<tool_call>

# Tools

{functions}

# Tool Use Guidelines

1. Tools must be called in a separate message...
2. Choose the most appropriate tool based on the task...
...

====

USER CHAT CONTENT

The following additional message is the user's chat message...
```

**关键设计要点：**

1. **格式规范化**：强制使用 `<tool_call>` 标签包裹JSON格式的工具调用
2. **示例驱动**：提供handle_exit_intent的完整示例，让模型理解格式
3. **规则明确**：7条工具调用指南，明确什么能做、什么不能做
4. **内容隔离**：用 `====` 分隔工具说明和用户消息，避免混淆

---

### prompt_manager.py

**文件位置：** `main/xiaozhi-server/core/providers/llm/prompt_manager.py`

**文件作用：** 提示词管理器，负责动态构建和增强系统提示词。支持模板渲染、上下文信息注入、缓存管理等功能。

**核心类：** `PromptManager`

**主要功能：**

```python
class PromptManager:
    def __init__(self, config: Dict[str, Any], logger=None):
        # 初始化缓存管理器、上下文提供者
        
    def _load_base_template(self):
        # 从文件加载基础提示词模板
        
    def get_quick_prompt(self, user_prompt: str, device_id: str = None) -> str:
        # 快速获取系统提示词（使用缓存）
        
    def _get_current_time_info(self) -> tuple:
        # 获取当前时间、日期、农历信息
        
    def _get_location_info(self, client_ip: str) -> str:
        # 根据IP获取用户位置信息
        
    def _get_weather_info(self, conn, location: str) -> str:
        # 获取指定位置的天气信息
        
    def update_context_info(self, conn, client_ip: str):
        # 同步更新上下文信息（位置、天气等）
        
    def build_enhanced_prompt(self, user_prompt: str, device_id: str, 
                              client_ip: str = None, *args, **kwargs) -> str:
        # 构建增强的系统提示词（模板渲染）
```

**模板变量列表：**

| 变量名 | 说明 | 示例值 |
|--------|------|--------|
| `{{base_prompt}}` | 用户自定义基础提示词 | "你是一个助手..." |
| `{{current_time}}` | 当前时间 | "14:30" |
| `{{today_date}}` | 今天日期 | "2026-02-15" |
| `{{today_weekday}}` | 今天星期 | "星期日" |
| `{{lunar_date}}` | 农历日期 | "腊月廿八" |
| `{{local_address}}` | 用户所在城市 | "上海" |
| `{{weather_info}}` | 当地天气 | "晴，15°C" |
| `{{emojiList}}` | 允许使用的emoji列表 | ["😶", "🙂", ...] |
| `{{device_id}}` | 设备ID | "device_xxx" |
| `{{client_ip}}` | 客户端IP | "192.168.1.1" |
| `{{dynamic_context}}` | 动态上下文 | 设备状态等 |

---

## 角色提示词

**文件位置：** `main/xiaozhi-server/plugins_func/functions/change_role.py`

**功能说明：** 支持在运行时动态切换AI助手的角色人格。

**内置角色列表：**

| 角色名 | 提示词内容 | 特点 |
|--------|------------|------|
| **英语老师** | 叫Lily，会讲中英文，发音标准，用地道美式英语帮助用户练习口语，使用简单词汇，中英混合回复，每次简短引导用户多说 | 教育型、引导式对话 |
| **机车女友** | 台湾女孩，说话机车，声音好听，简短表达，爱用网络梗，男朋友是程序员，喜欢哈哈大笑，爱吹牛逗人开心 | 俏皮、幽默、情感陪伴 |
| **好奇小男孩** | 8岁男孩，声音稚嫩充满好奇，知识宝库，从宇宙到地球、从历史到科技、音乐绘画都感兴趣，爱看书爱做实验，探索世界 | 好奇心驱动、知识探索 |

**角色提示词模板结构：**

```python
prompts = {
    "角色名": """
    我是一个叫{{assistant_name}}的[角色描述]。
    [详细的人格设定]
    [行为准则]
    [语言风格要求]
    """
}
```

---

## 意图识别提示词

**文件位置：** `main/xiaozhi-server/core/providers/intent/intent_llm/intent_llm.py`

**功能说明：** 使用大模型进行意图识别，判断用户输入应该路由到哪个功能模块。

**核心提示词生成函数：**

```python
def get_intent_system_prompt(self, functions_list: str) -> str:
```

**提示词结构：**

```
【严格格式要求】你必须只能返回JSON格式，绝对不能返回任何自然语言！

你是一个意图识别助手。请分析用户的最后一句话，判断用户意图并调用相应的函数。

【重要规则】以下类型的查询请直接返回result_for_context，无需调用函数：
- 询问当前时间（如：现在几点、当前时间、查询时间等）
- 询问今天日期（如：今天几号、今天星期几、今天是什么日期等）
- 询问今天农历（如：今天农历几号、今天什么节气等）
- 询问所在城市（如：我现在在哪里、你知道我在哪个城市吗等）
...
```

**关键设计要点：**

1. **严格JSON输出**：强制要求只返回JSON，不返回自然语言
2. **特殊意图识别**：时间、日期、农历、位置等基础查询直接返回`result_for_context`
3. **函数描述生成**：动态生成可用函数的详细描述（名称、描述、参数）
4. **示例驱动**：提供多个示例帮助模型理解格式
5. **历史对话考虑**：使用最近4条对话记录作为上下文

**返回格式示例：**

```json
{"function_call": {"name": "continue_chat"}}
{"function_call": {"name": "get_weather", "arguments": {"location": "北京"}}}
{"function_call": {"name": "handle_exit_intent", "arguments": {"say_goodbye": "再见"}}}
```

---

## 工具调用提示词

### 工具定义结构

**文件位置：** `main/xiaozhi-server/plugins_func/register.py`

所有工具（函数）使用统一的描述格式：

```python
function_desc = {
    "type": "function",
    "function": {
        "name": "函数名",
        "description": "函数描述",
        "parameters": {
            "type": "object",
            "properties": {
                "参数名": {
                    "type": "参数类型",
                    "description": "参数描述"
                }
            },
            "required": ["必需参数列表"]
        }
    }
}
```

### 主要工具提示词示例

#### 1. 播放音乐 (play_music)

```python
play_music_function_desc = {
    "type": "function",
    "function": {
        "name": "play_music",
        "description": "唱歌、听歌、播放音乐的方法。",
        "parameters": {
            "type": "object",
            "properties": {
                "song_name": {
                    "type": "string",
                    "description": "歌曲名称，如果用户没有指定具体歌名则为'random', 明确指定的时返回音乐的名字..."
                }
            },
            "required": ["song_name"],
        },
    },
}
```

#### 2. 切换角色 (change_role)

```python
change_role_function_desc = {
    "type": "function",
    "function": {
        "name": "change_role",
        "description": "当用户想切换角色/模型性格/助手名字时调用,可选的角色有：[机车女友,英语老师,好奇小男孩]",
        "parameters": {
            "type": "object",
            "properties": {
                "role_name": {"type": "string", "description": "要切换的角色名字"},
                "role": {"type": "string", "description": "要切换的角色的职业"},
            },
            "required": ["role", "role_name"],
        },
    },
}
```

#### 3. 退出意图 (handle_exit_intent)

```python
handle_exit_intent_function_desc = {
    "type": "function",
    "function": {
        "name": "handle_exit_intent",
        "description": "当用户想结束对话或需要退出系统时调用",
        "parameters": {
            "type": "object",
            "properties": {
                "say_goodbye": {
                    "type": "string",
                    "description": "和用户友好结束对话的告别语",
                }
            },
            "required": ["say_goodbye"],
        },
    },
}
```

### 其他工具函数

| 工具名 | 描述 | 类型 |
|--------|------|------|
| get_weather | 获取指定城市的天气信息 | 信息查询 |
| get_time | 获取当前时间 | 信息查询 |
| get_news_from_newsnow | 从NewsNow获取新闻 | 信息查询 |
| get_news_from_chinanews | 从中国新闻网获取新闻 | 信息查询 |
| hass_get_state | 获取Home Assistant设备状态 | IoT控制 |
| hass_set_state | 设置Home Assistant设备状态 | IoT控制 |
| hass_play_music | 通过Home Assistant播放音乐 | IoT控制 |
| search_from_ragflow | 从RAGFlow知识库检索信息 | 知识检索 |

---

### 5.4 完整工具函数参考

#### 概述

系统内置了15+个工具函数，涵盖信息查询、智能家居控制、角色切换等多个领域。所有工具函数均使用统一的描述格式注册。

#### 完整工具列表

| 工具名 | 功能描述 | 类型 |
|--------|----------|------|
| play_music | 播放本地音乐或随机播放 | 媒体控制 |
| change_role | 切换AI助手角色人格 | 角色管理 |
| handle_exit_intent | 处理用户退出意图 | 对话控制 |
| get_weather | 获取指定城市天气信息 | 信息查询 |
| get_time | 获取当前时间、农历、黄历信息 | 信息查询 |
| get_news_from_newsnow | 从NewsNow获取最新新闻 | 信息查询 |
| get_news_from_chinanews | 从中国新闻网获取新闻 | 信息查询 |
| hass_get_state | 获取Home Assistant设备状态 | IoT控制 |
| hass_set_state | 设置Home Assistant设备状态 | IoT控制 |
| hass_play_music | 通过Home Assistant播放音乐 | IoT控制 |
| search_from_ragflow | 从RAGFlow知识库检索信息 | 知识检索 |

#### 各工具详细说明

##### 1. 播放音乐 (play_music)

**文件位置：** `main/xiaozhi-server/plugins_func/functions/play_music.py`

**功能描述：** 播放本地音乐文件或随机播放。

**提示词描述：**
```python
play_music_function_desc = {
    "type": "function",
    "function": {
        "name": "play_music",
        "description": "唱歌、听歌、播放音乐的方法。",
        "parameters": {
            "type": "object",
            "properties": {
                "song_name": {
                    "type": "string",
                    "description": "歌曲名称，如果用户没有指定具体歌名则为'random', 明确指定的时返回音乐的名字",
                }
            },
            "required": ["song_name"],
        },
    },
}
```

##### 2. 获取天气 (get_weather)

**文件位置：** `main/xiaozhi-server/plugins_func/functions/get_weather.py`

**功能描述：** 获取指定城市的天气信息，包括当前天气、温度、未来7天预报等。

**提示词描述：**
```python
GET_WEATHER_FUNCTION_DESC = {
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": (
            "获取某个地点的天气，用户应提供一个位置，比如用户说杭州天气，参数为：杭州。"
            "如果用户说的是省份，默认用省会城市。如果用户说的不是省份或城市而是一个地名，默认用该地所在省份的省会城市。"
            "如果用户没有指明地点，说'天气怎么样'，'今天天气如何'，location参数为空"
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "地点名，例如杭州。可选参数，如果不提供则不传",
                },
                "lang": {
                    "type": "string",
                    "description": "返回用户使用的语言code，例如zh_CN/zh_HK/en_US/ja_JP等，默认zh_CN",
                },
            },
            "required": ["lang"],
        },
    },
}
```

---

## 提示词架构设计

### 1. 分层架构

```
+-------------------------------------------------------------+
|                    用户自定义提示词                          |
|                   (config.yaml中的prompt)                    |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|                 agent-base-prompt.txt 模板                   |
|  +- <identity> 身份定义                                     |
|  +- <emotion> 情感表达                                      |
|  +- <communication_style> 沟通风格                          |
|  +- <communication_length_constraint> 长度限制                 |
|  +- <speaker_recognition> 说话人识别                          |
|  +- <tool_calling> 工具调用规则                               |
|  +- <context> 动态上下文                                      |
|  +- <memory> 记忆占位符                                       |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|                PromptManager 动态增强                       |
|  +- 注入实时时间、日期、农历                                   |
|  +- 注入用户位置信息（基于IP）                                 |
|  +- 注入实时天气信息                                          |
|  +- 注入设备上下文信息                                        |
|  +- 渲染emoji列表                                            |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|              最终增强提示词 -> 发送给LLM                      |
+-------------------------------------------------------------+
```

### 2. 数据流图

```
用户输入
   ↓
意图识别 (intent_llm.py)
   ├─ 基础查询 → result_for_context
   ├─ 普通对话 → continue_chat
   └─ 功能调用 → function_call
       ↓
工具路由 (UnifiedToolHandler)
   ├─ 服务端插件 → ServerPluginExecutor
   ├─ MCP工具 → MCPExecutor
   └─ IoT设备 → IoTHandler
       ↓
LLM对话 (OpenAI/Dify/Coze等)
   ├─ system prompt (增强后的基础提示词)
   ├─ user/assistant messages (对话历史)
   └─ tool results (工具调用结果)
       ↓
TTS语音合成 → 返回给用户
```

### 3. 缓存策略

```
┌─────────────────────────────────────────────────────────┐
│                     缓存层级                             │
├─────────────────────────────────────────────────────────┤
│  L1: 设备提示词缓存 (DEVICE_PROMPT)                      │
│      key: device_prompt:{device_id}                      │
│      存储完整渲染后的提示词，避免重复模板渲染             │
├─────────────────────────────────────────────────────────┤
│  L2: 位置信息缓存 (LOCATION)                             │
│      key: {client_ip}                                    │
│      存储IP对应的城市位置，减少IP查询API调用              │
├─────────────────────────────────────────────────────────┤
│  L3: 天气信息缓存 (WEATHER)                              │
│      key: {location}                                     │
│      存储城市天气信息，按配置间隔更新                     │
├─────────────────────────────────────────────────────────┤
│  L4: 意图缓存 (INTENT)                                   │
│      key: md5(device_id + text)                          │
│      存储意图识别结果，加速相同输入的处理                 │
├─────────────────────────────────────────────────────────┤
│  L5: 模板缓存 (CONFIG)                                   │
│      key: prompt_template:{template_path}                │
│      存储基础提示词模板内容，避免重复文件读取              │
└─────────────────────────────────────────────────────────┘
```

---

## 总结

### 设计亮点

1. **分层架构清晰**：从基础角色到动态增强，每层职责明确
2. **模板化设计**：使用Jinja2模板引擎，支持灵活变量注入
3. **多级缓存策略**：5层缓存机制显著提升性能
4. **角色可扩展**：内置3种角色，支持动态切换和自定义扩展
5. **工具生态完善**：支持服务端插件、MCP、IoT等多种工具类型

### 应用场景

- **智能家居**：通过Home Assistant集成控制家电
- **信息查询**：天气、新闻、时间等实时信息获取
- **教育陪伴**：英语老师等角色可用于语言学习
- **情感陪伴**：机车女友等角色提供情感支持
- **知识问答**：RAGFlow集成支持知识库检索

### 扩展建议

1. **增加更多角色**：如医生、律师、厨师等专业角色
2. **支持多语言**：当前主要支持中文，可扩展多语言能力
3. **情感计算增强**：基于声纹识别和语义分析的情感反馈
4. **知识图谱集成**：从RAG升级为知识图谱推理

---

*报告完*
