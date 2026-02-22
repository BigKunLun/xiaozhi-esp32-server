# MCP 协议与多执行器架构深度分析 Spec

## Why

小智 ESP32 服务器项目是一个典型的 AI Agent 架构实现，其核心特点是：
1. 大量使用 MCP (Model Context Protocol) 作为工具调用协议
2. 实现了多种 Tool Executor 来支持不同的工具类型
3. 通过 Unified Tool Handler 统一管理所有工具调用

深入分析这些实现，可以帮助我们：
- 理解 MCP 协议在实际项目中的落地方式
- 学习多执行器架构的设计模式
- 掌握 Agent 系统的工具管理最佳实践

## What Changes

本次分析将产出以下交付物：
1. **MCP 协议实现分析报告** - 分析项目中 MCP 的完整实现流程
2. **多执行器架构分析报告** - 分析 5 种 Executor 的设计与协作
3. **架构设计最佳实践总结** - 提炼可复用的设计模式

## Impact

- **Affected Code**: 
  - `core/providers/tools/server_mcp/` - 服务端 MCP 实现
  - `core/providers/tools/device_mcp/` - 设备端 MCP 实现
  - `core/providers/tools/mcp_endpoint/` - MCP 接入点
  - `core/providers/tools/server_plugins/` - 服务端插件
  - `core/providers/tools/device_iot/` - 设备 IoT
  - `core/providers/tools/unified_tool_*.py` - 统一工具管理

## ADDED Requirements

### Requirement: MCP 协议实现分析

The system SHALL provide a comprehensive analysis of MCP protocol implementation including:

#### Scenario: Server MCP Analysis
- **WHEN** analyzing `server_mcp/mcp_manager.py` and `server_mcp/mcp_executor.py`
- **THEN** identify:
  - How MCP servers are initialized and managed
  - How tool calls are executed via MCP protocol
  - How responses are processed and returned

#### Scenario: Device MCP Analysis
- **WHEN** analyzing `device_mcp/mcp_client.py` and `device_mcp/mcp_handler.py`
- **THEN** identify:
  - How device-side MCP client manages tool state
  - How WebSocket is used for MCP communication
  - How tool calls are sent and responses received

#### Scenario: MCP Endpoint Analysis
- **WHEN** analyzing `mcp_endpoint/mcp_endpoint_executor.py` and `mcp_endpoint/mcp_endpoint_client.py`
- **THEN** identify:
  - How external MCP endpoints are connected
  - How HTTP POST is used for tool execution
  - How base64 encoding is handled in responses

### Requirement: 多执行器架构分析

The system SHALL provide a comprehensive analysis of the multi-executor architecture including:

#### Scenario: Base Architecture Analysis
- **WHEN** analyzing `base/tool_executor.py` and `base/tool_types.py`
- **THEN** identify:
  - The abstract base class design for ToolExecutor
  - The ToolResponse and ToolDefinition data structures
  - The ContentType enumeration

#### Scenario: Unified Tool Management Analysis
- **WHEN** analyzing `unified_tool_manager.py` and `unified_tool_handler.py`
- **THEN** identify:
  - How ToolManager coordinates multiple executors
  - How tools are registered and discovered
  - How function calls are routed to appropriate executors

#### Scenario: Individual Executor Analysis
- **WHEN** analyzing each of the 5 executors
- **THEN** identify for each:
  - Its specific responsibility and use case
  - How it implements the ToolExecutor interface
  - Its unique characteristics and design decisions

### Requirement: 架构设计最佳实践总结

The system SHALL provide a summary of architectural best practices including:

#### Scenario: Design Patterns Identification
- **WHEN** reviewing the complete architecture
- **THEN** identify and document:
  - Strategy Pattern (different executors)
  - Factory Pattern (tool creation)
  - Observer Pattern (event handling)
  - Adapter Pattern (MCP protocol adaptation)

#### Scenario: Extensibility Analysis
- **WHEN** reviewing how new tool types are added
- **THEN** identify:
  - How the plugin system works
  - How new executors can be registered
  - How tool definitions are discovered

#### Scenario: Error Handling Strategy
- **WHEN** reviewing error handling across executors
- **THEN** identify:
  - Common error handling patterns
  - How errors are propagated
  - How ActionResponse is used for error communication
