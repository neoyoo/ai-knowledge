---
title: "Agent Registry & Discovery — AgentScope Java"
category: L2
parent: "[[agent-registry-discovery]]"
source: agentscope-java
source_version: "2.0.0-SNAPSHOT"
confidence: high
created: 2026-06-09
updated: 2026-06-10
relations:
  - target: "[[agent-registry-discovery]]"
    type: implements
  - target: "[[channel-remote]]"
    type: feeds
  - target: "[[multi-agent]]"
    type: feeds
---

## 概述

AgentScope Java 是独立于 Python AgentScope 2.x 的新源，不是 Python 仓库的子版本。它把 AgentCard / A2A / Nacos / Spring Boot starter 做成企业 Java 服务化链路：agent 既可以作为 Reactor runtime 中的本地对象运行，也可以通过 A2A JSON-RPC/SSE 暴露为可发现、可注册、可被跨进程调用的服务。

这才是当前 AgentScope 生态中 AgentCard + Nacos 的源码-backed 主参考；Python 2.x 当前主线是 session/workspace/team runtime，不应继续把 A2A/Nacos 归给 Python 源码。

## 架构

```text
ReActAgent / HarnessAgent
  │
  ├─ AgentStateStore(userId, sessionId)
  │    ├─ JsonFile
  │    └─ Redis / MySQL / OSS extensions
  │
  ├─ A2A Server
  │    ├─ AgentCard endpoint /.well-known/...
  │    ├─ JSON-RPC / SSE controller
  │    └─ AgentRegistry abstraction
  │
  ├─ A2A Client
  │    ├─ WellKnownAgentCardResolver
  │    └─ NacosAgentCardResolver
  │
  └─ Spring Boot Starter
       └─ AutoConfiguration wires server, controllers, ready listener
```

## 关键代码证据

### Reactor agent runtime

- `agentscope-core/.../ReActAgent.java:153` — 基于 Project Reactor streaming
- `ReActAgent.java:259` — 按 `(userId, sessionId)` slot 缓存 state 和 permission engine
- `ReActAgent.java:475` — 同 slot 串行、不同 session 并行

### State store

- `AgentStateStore.java:22` — 持久化接口以 `(userId, sessionId)` 为 key
- `JsonFileAgentStateStore.java:41` — per-user layout + atomic/hash append
- `RedisAgentStateStore.java:37` — Redis 扩展支持 Jedis/Lettuce/Redisson

### A2A / AgentCard

- `A2aAgent.java:54` — Java client 侧 A2A agent
- `AgentCardResolver.java:21` — AgentCard resolver 抽象
- `WellKnownAgentCardResolver.java:23` — well-known URL resolver
- `AgentRegistry.java:23` — server 侧 AgentRegistry 抽象
- `AgentScopeA2aServer.java:155` — endpoint ready 后自动注册 AgentCard

### Nacos

- `NacosAgentCardResolver.java:90` — subscribe AgentCard 并缓存更新
- `NacosA2aRegistry.java:68` — release AgentCard 并注册 endpoint

### Spring Boot starter

- `AgentscopeA2aAutoConfiguration.java:84` — 自动创建 `AgentScopeA2aServer`、AgentCard controller、JSON-RPC controller、ready listener
- `AgentCardController.java:36` — well-known AgentCard endpoint
- `A2aJsonRpcController.java:50` — JSON-RPC / SSE endpoint

## 设计亮点

### 1. Java 企业服务化路径完整

Python 生态常把 agent registry 停在内存对象或 FastAPI router。AgentScope Java 直接接入 Spring Boot auto-configuration、Nacos registry/resolver、JSON-RPC/SSE controller，适合企业 Java 基建。

### 2. AgentCard 是运行时注册对象

AgentCard 不只是文档；server ready 后会注册 AgentCard，client 端 resolver 能订阅和缓存更新。Nacos 负责动态发现和 endpoint 管理。

### 3. `(userId, sessionId)` 贯穿 state 与并发

ReActAgent 按 slot 缓存 state 和 permission engine，同 slot 串行，不同 session 并行。这是 agent 领域 session affinity 的 Java 实现，对 prompt cache、状态一致性和并发控制都有直接意义。

## 局限

- 这是 Java/Spring/Nacos 生态的生产路径；如果 agent-os 的 P1 只是单机或简单 Web worker，直接上 Nacos 会过重。
- AgentCard 能描述能力和 endpoint，但能力匹配、调度策略、负载均衡和版本治理仍需要上层策略。

## 对 agent-os 的借鉴

agent-os 的 registry 可以分两档：

- P1：自建轻量 registry（Postgres/Redis/k8s Service），解决 worker endpoint、heartbeat、session affinity。
- P2/P3：引入 AgentCard/A2A/Nacos 风格，用于跨组织/跨系统 agent 互操作。

AgentScope Java 的价值在于给第二档提供完整参考，而不是要求所有 agent-os 部署都引入 Nacos。

## 来源

- 项目：`/Users/neo/Desktop/project/git/agentscope-java`
- 版本：`2.0.0-SNAPSHOT`
