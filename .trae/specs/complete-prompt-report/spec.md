# 补充提示词分析报告缺失内容 Spec

## Why

现有的 `prompt_analysis_report.md` 虽然涵盖了核心的提示词架构，但存在以下缺失：
1. 工具函数描述不完整 - 只列举了3个工具，实际有15+个工具
2. MCP 工具提示词处理逻辑未提及
3. IoT 设备提示词动态生成机制未说明
4. Memory 模块如何注入提示词未深入
5. 动态上下文的详细内容未展开

需要补充这些缺失内容，使报告成为完整的提示词文档。

## What Changes

本次任务将补充以下内容到报告中：
1. **完整工具函数列表** - 补充所有15+个工具函数的提示词描述
2. **MCP 工具提示词处理** - 说明 MCP 工具如何生成和使用提示词
3. **IoT 设备提示词机制** - 解释 IoT 设备描述如何转换为提示词
4. **Memory 注入机制** - 说明历史对话如何注入到提示词中
5. **动态上下文详细说明** - 展开 `{{dynamic_context}}` 包含的具体内容

**原则：只添加缺失内容，不修改已有内容**

## Impact

- **Affected Document**: `prompt_analysis_report.md`
- **新增章节**:
  - 7.1 完整工具函数参考
  - 7.2 MCP 工具提示词处理
  - 7.3 IoT 设备提示词机制
  - 7.4 Memory 注入机制
  - 7.5 动态上下文详细说明

## ADDED Requirements

### Requirement: 补充完整工具函数列表

The system SHALL provide a comprehensive list of all tool functions in the report.

#### Scenario: List all tools
- **WHEN** reading the tools section
- **THEN** the following tools MUST be documented:
  - play_music
  - change_role
  - handle_exit_intent
  - get_weather
  - get_time
  - get_news_from_newsnow
  - get_news_from_chinanews
  - hass_get_state
  - hass_set_state
  - hass_play_music
  - search_from_ragflow

### Requirement: 补充 MCP 工具提示词处理

The system SHALL explain how MCP tools generate and use prompts.

#### Scenario: MCP prompt handling
- **WHEN** reading the MCP section
- **THEN** it MUST explain:
  - How MCP tool descriptions are converted to prompt format
  - How MCP server capabilities are injected into prompts
  - The difference between device MCP and server MCP prompt handling

### Requirement: 补充 IoT 设备提示词机制

The system SHALL explain how IoT device descriptions are converted to prompts.

#### Scenario: IoT prompt mechanism
- **WHEN** reading the IoT section
- **THEN** it MUST explain:
  - How device descriptors are parsed
  - How device capabilities are formatted as tool descriptions
  - How device state is included in context

### Requirement: 补充 Memory 注入机制

The system SHALL explain how historical dialogue is injected into prompts.

#### Scenario: Memory injection
- **WHEN** reading the memory section
- **THEN** it MUST explain:
  - How conversation history is retrieved
  - How history is formatted and injected into the `<memory>` tag
  - The role of different memory providers (mem0ai, mem_local_short, powermem)

### Requirement: 补充动态上下文详细说明

The system SHALL provide detailed explanation of `{{dynamic_context}}` content.

#### Scenario: Dynamic context details
- **WHEN** reading the dynamic context section
- **THEN** it MUST list:
  - What device information is included
  - What session state is tracked
  - How context data is fetched and formatted
  - The role of ContextDataProvider

## REMOVED Requirements

None - This is an additive task, no content will be removed.

## MODIFIED Requirements

None - Existing content will not be modified, only new content will be added.