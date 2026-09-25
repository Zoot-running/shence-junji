# CAI（Cybersecurity AI）黑盒 SRC 分析报告

> 评估场景：只有网站、API、域名、IP、测试账号和授权范围，没有目标源码。
>
> **综合评分：45/100。定位：有专用 Bug Bounty 方法论的通用 LLM Agent 框架。**

## 1. 结论

CAI 是少数在角色和提示词层明确面向 bug bounty 的项目。它有 bug_bounter、web_pentester、retester、reporter，以及发现与复测循环 handoff；在业务逻辑推理方面优于纯扫描器。

但实际工具层很薄：主要是通用 shell、代码执行、Shodan、Web 情报和可选 C99。没有真实浏览器、MITM 代理、稳定认证态和强制证据存档。项目已归档，也限制了长期使用价值。

## 2. 主要架构

- Python，多 Agent 框架，基于 Agent、Tools、Handoffs、Patterns、Turns、Tracing、Guardrails 和 HITL。
- CLI 入口位于 `src/cai/cli.py`。
- `src/cai/agents/bug_bounter.py` 定义漏洞赏金 Agent。
- `src/cai/agents/retester.py` 定义独立复测 Agent。
- `src/cai/agents/reporter.py` 只生成报告，不拥有执行工具。
- `src/cai/agents/patterns/bb_triage.py` 在发现和复测 Agent 之间循环 handoff。

## 3. 黑盒能力

### 3.1 侦察

bug_bounter 实际拥有：

- `generic_linux_command`；
- `execute_code`；
- Shodan 搜索与主机信息；
- Web 情报工具；
- 可选 Google/C99/子域枚举。

Agent 可以通过 shell 手工运行 nmap、ffuf、gobuster、sqlmap 等，但这些通常不是专用 wrapper，是否可用取决于本机安装。

### 3.2 浏览器、代理与认证态

CAI 的 Web 工具主要是 fetch URL、headers 和 search，不提供：

- JS 浏览器；
- 表单点击/输入；
- 持久 Cookie jar 和多角色会话；
- HTTP MITM、请求历史和重放；
- 截图证据。

这使它不适合复杂登录态和前端业务流程。

## 4. 业务逻辑与复测

`system_bug_bounter.md` 明确要求：

- 定义域名、子域和 IP 范围；
- 枚举 URL、API 和角色；
- 测试认证、授权、业务逻辑和竞态；
- 记录复现、影响和修复；
- 避免破坏性测试和数据外传。

`bb_triage.py` 会把候选漏洞交给 Retester，再返回 Bug Bounter 继续发现。这是合理的反误报组织方式，但验证仍依赖相同模型家族、通用 shell 和会话文本。

## 5. 证据与报告

Reporter 没有执行工具，只根据会话生成 HTML，可避免报告阶段继续执行攻击。缺点是：

- finding 没有强 schema；
- 无 HTTP exchange ID；
- 无截图/原始请求自动归档；
- evidence 主要是自然语言和 shell 输出；
- 缺漏洞生命周期和跨轮去重数据库。

## 6. Scope、安全与部署

Prompt 要求遵守 scope 和非破坏原则，guardrail 主要用于 prompt injection 和命令风险，不是网络层 egress allowlist。没有统一限速、并发和 risk-tier 审批门。

部署依赖 Python、LLM key、外部 CLI 和可选情报 API。项目已归档，不再提供修复、兼容和安全更新，默认/推荐模型的一部分能力也迁移到后续商业系统。

## 7. README 与实现差异

- “bug-bounty-ready”和历史 CVE 成绩不能直接证明任意网站的稳定发现率。
- 文档列出的 ffuf/sqlmap/nuclei 等多是 Agent 在 shell 中调用，不是仓库内封装工具。
- 利用、提权和横移的专用工具远少于整体宣传给人的印象。

## 8. 评分

| 维度 | 分数 |
|---|---:|
| 资产/子域/API 发现 | 65 |
| 浏览器/代理/认证态 | 20 |
| 业务逻辑 | 60 |
| 自动利用与 PoC | 45 |
| 误报控制 | 35 |
| 证据与报告 | 35 |
| Scope/限速/审批 | 40 |
| 多目标与扩展 | 55 |
| 部署成本 | 35 |

**最终判断：45/100。** 可借鉴 bug bounty/retester/reporter 的角色设计，也可作为业务逻辑推理 Agent；不能单独承担认证态 Web SRC 全流程。

## 9. 关键源码依据

- `src/cai/cli.py`
- `src/cai/agents/bug_bounter.py`
- `src/cai/agents/retester.py`
- `src/cai/agents/reporter.py`
- `src/cai/agents/patterns/bb_triage.py`
- `src/cai/prompts/system_bug_bounter.md`
- `src/cai/prompts/system_triage_agent.md`
- `src/cai/tools/reconnaissance/generic_linux_command.py`
- `src/cai/tools/web/`
- `src/cai/agents/guardrails.py`
