# T3MP3ST 分析报告

- **能力范围**：全流程
- **SRC 适用度**：中——黑盒 web 侦察/扫描稳定且 XBEN 90.1% 可复现，白盒 CVE-zero 8/10 精确到文件行/CWE，直指真实源码审计与黑盒 SRC；但利用与蜂群未 benchmark、下游仅实验。

## 核心思路
把用户本机已登录的 AI 编程 agent(Claude Code/Codex/本地 Ollama 等)当作大脑，用 ReAct 循环驱动内置 Arsenal 工具库与 8 角色杀伤链，自主完成侦察→利用→验证→报告的黑/白盒挖洞，且所有 benchmark 可复现(verify-claims)。

## 架构
无 key 本地 agent 接入，统一 LLM Backbone 驱动单 Agent ReAct 循环；8 个 MITRE 杀伤链 operator 由 MissionControl+TaskQueue 调度到 OperatorCell；PackBoard 共享线索板蜂群协同；证据门+Arsenal scope/审批围栏；白盒走 DecompositionOrchestrator 双模型分解。

## 技术面
黑盒 ReAct+函数调用(36 内置工具，opt-in 111 适配器)；白盒 web-tree-sitter 多语言摄入+安全优先级排序+盲主构建者双模型分解；8-operator 多 agent 蜂群+共享线索板(认领/背书/驳斥去重)；Playwright 无头浏览器前端检测；证据溯源门+refuter 反证面板+MCP 接入。

## 短板
下游利用/渗透 operator 仅实验、蜂群无端到端验证；白盒非 Python 语言 fail-open 甚至漏提取；分解编排更耗 token；高级模块仍为 stub；模型断言 finding 被强制降级到 low。

## 值得神策借鉴
证据溯源门(无真实工具输出的 finding 不标 verified、模型断言降级)杜绝幻觉高危；白盒'盲主构建者'分解绕安全对齐；出站 scope 围栏+dry_run 默认+verify-claims 可复现基准。
