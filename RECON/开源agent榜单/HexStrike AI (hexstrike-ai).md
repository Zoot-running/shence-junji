# HexStrike AI 黑盒 SRC 分析报告

> 评估场景：只有网站、API、域名、IP、测试账号和授权范围，没有目标源码。
>
> **综合评分：50/100。定位：工具覆盖很广但缺治理和推理的 MCP 安全工具网关。**

## 1. 结论

HexStrike AI 的资产、参数和 API 工具覆盖在十个项目中非常突出。它把 150+ 外部工具通过 MCP/HTTP 暴露给 Claude、Cursor 等外部 LLM，适合做大规模前置侦察和已知漏洞初筛。

但项目内部没有真实 LLM 推理，所谓 autonomous agents 和 intelligent decision engine 主要是硬编码打分、关键词和模板。更严重的是缺 scope、限速、审批和可靠报告，Flask 服务及 `shell=True` 执行也扩大了自身风险。它不适合无监管地自主运行真实 SRC。

## 2. 主要架构

- `hexstrike_mcp.py` 使用 FastMCP 暴露约 151 个工具。
- 工具调用通过 HTTP 转发到 Flask `hexstrike_server.py`。
- 服务端使用 subprocess 执行 nmap、nuclei、sqlmap、subfinder、amass 等外部命令。
- 内含线程/进程管理、LRU 缓存、浏览器 Agent 和可视化输出。
- LLM 位于外部 MCP 客户端，项目自身不负责模型推理。

## 3. 黑盒能力

### 3.1 资产、参数和 API

覆盖包括：

- amass、subfinder、httpx、naabu、rustscan；
- katana、gau、waybackurls、ffuf、gobuster；
- arjun、paramspider、x8；
- nuclei、nikto、sqlmap、Dalfox；
- GraphQL scanner、JWT analyzer、API fuzzer、API schema analyzer；
- Selenium BrowserAgent。

工具面非常适合从域名扩展到子域、URL、参数和已知漏洞。

### 3.2 业务逻辑和认证态

Selenium 能做部分页面交互和截图，但没有成熟的多角色会话、MFA、代理流量、请求重放和业务状态模型。内部“AI”不能理解复杂业务：

- decision engine 是固定工具分值；
- payload generator 是字符串模板；
- `ai_test_payload` 使用子串匹配；
- zero-day research 对 `source_code_url` 返回 `analysis_status: simulated`。

因此业务逻辑发现能力主要来自外部 LLM，而不是 HexStrike 本身。

## 4. 漏洞验证与报告

nuclei、sqlmap、Dalfox 等能真实执行并产生输出，但 HexStrike 缺：

- 统一复测 Agent；
- counterevidence；
- 强制 PoC/影响 finding schema；
- HTTP exchange 证据；
- 漏洞生命周期和去重；
- 可直接提交的标准 SRC 报告。

ModernVisualEngine 的输出主要用于终端展示，不能替代完整报告。

## 5. Scope、安全和部署风险

- 没有可靠目标 allowlist、重定向检查和网络 egress 门。
- 没有 risk-tier 审批和 SRC 速率策略。
- Flask 默认监听 `0.0.0.0:8888`，缺成熟鉴权。
- 部分命令通过 `shell=True` 构造，参数处理不当会形成自身命令注入风险。
- 150+ 工具多数需用户单独安装，不是 pip requirements 自动提供。

## 6. README 与实现差异

- “12+ autonomous AI agents”主要是硬编码类和流程模板，项目内部无 OpenAI/Anthropic/LiteLLM 模型调用。
- 工具数包含重复 wrapper，数量不等于独立能力。
- “98.7% 检测率”和 bug bounty 成绩缺本地测量与可复核 benchmark。
- “高级报告”主要是可视化和 API 返回，不是 SRC finding 管理。

## 7. 评分

| 维度 | 分数 |
|---|---:|
| 资产/子域/API 发现 | 85 |
| 浏览器/代理/认证态 | 40 |
| 业务逻辑 | 15 |
| 自动利用与 PoC | 55 |
| 误报控制 | 30 |
| 证据与报告 | 45 |
| Scope/限速/审批 | 10 |
| 多目标与扩展 | 40 |
| 部署成本 | 30 |

**最终判断：50/100。** 适合作为外部 Agent 的侦察和工具网关；必须放在独立 sandbox、网络 scope 网关、审批和报告系统之后使用。

## 8. 关键源码依据

- `hexstrike_mcp.py`
- `hexstrike_server.py`
- `requirements.txt`
- `hexstrike_server.py` 中 `IntelligentDecisionEngine`
- `hexstrike_server.py` 中 payload generator 和 `ai_test_payload`
- `hexstrike_server.py` 中 `zero_day_research`
- `hexstrike_server.py` 中 `ModernVisualEngine`
