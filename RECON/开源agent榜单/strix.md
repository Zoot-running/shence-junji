# Strix 黑盒 SRC 分析报告

> 评估场景：只有网站、API、域名、IP、测试账号和授权范围，没有目标源码。
>
> **综合评分：70/100。定位：最适合直接做认证态 Web/API SRC 的实战型 Agent。**

## 1. 结论

Strix 的总分略低于 CyberStrikeAI 和 T3MP3ST，但如果目标是直接测试网站/API，它是十个项目中最完整、最接近日常安全研究员工作流的一个。核心原因是：真实浏览器、Caido MITM 代理、认证态请求、OpenAPI 输入、外部 CLI、PoC、反证和多格式报告被放在同一流程里。

主要风险是 scope、限速和危险操作审批没有 T3MP3ST 那样的执行层硬门。系统提示中的授权目标很明确，但外部命令、重定向和间接访问仍需额外网关约束。

## 2. 主要架构

- Python 3.12，基于 `openai-agents` 和 LiteLLM。
- CLI/TUI 入口，支持本地 viewer 和 CI/CD。
- `strix/core/agents.py` 的 `AgentCoordinator` 管理 Agent 图、父子关系、mailbox、预算和快照。
- Docker 沙箱运行 shell、浏览器、扫描器和代理。
- Caido 负责 HTTP 拦截、历史、重放和证据。
- Skills 按场景、协议和漏洞类型动态加载。

## 3. 黑盒能力

### 3.1 资产和 API 发现

输入支持 URL、IP、OpenAPI/Postman、多 `-t` 和 `--target-list`。资产发现 Skill 覆盖：

- CT、被动 DNS、ASN；
- subfinder、httpx、naabu；
- katana、ffuf、nuclei、sqlmap；
- API schema 和端点测试。

适合从单网站扩展到一组子域、服务和 API。

### 3.2 浏览器、代理和认证态

- agent_browser 执行网页交互；
- Caido 记录和修改 HTTP 流量；
- 支持把账号、登录说明和特殊约束通过 instruction 传入；
- 可保存 Cookie/会话并在浏览器与代理间关联；
- 适合多角色 IDOR、授权绕过、表单和前端路径测试。

这部分是 Strix 相对其他开源项目最明显的优势。

## 4. 漏洞验证与证据

`strix/tools/reporting/tool.py` 的报告工具要求或支持：

- PoC 描述与 `poc_script_code`；
- evidence 与 counterevidence；
- confidence、CVSS 和修复建议；
- `http_exchange_ids`，并对当前 Caido 项目做校验。

输出包括逐 finding Markdown、JSON、CSV、PDF、SARIF 和本地 viewer。黑盒场景最有价值的是 HTTP exchange、PoC 和截图；SARIF 的源码定位在无源码时不是核心。

## 5. 业务逻辑能力

Deep mode 要求建立用户流程、状态机、信任边界、不变量和多角色模型，并按组件/功能/漏洞类派生子 Agent。它具备发现业务逻辑漏洞的工作方式，但完成度仍由模型和预算决定，没有确定性业务 oracle。

## 6. Scope 与运行安全

- 平台将 authorized targets 作为系统级上下文注入，普通用户文本不能扩展范围。
- 这能降低 prompt 层越界，但没有在每个网络工具执行前统一解析 egress 目标。
- 限速和非破坏性原则主要存在于 Skill/提示词。
- 缺统一的 risk-tier 审批门，高风险工具需要外围策略控制。

## 7. 部署成本

需要 Docker、LLM key 和较大的安全工具镜像；Caido 增加组件复杂度，但也提供了黑盒 SRC 最重要的代理证据。对单次简单扫描偏重，对持续 Web SRC 是合理成本。

## 8. README 与实现差异

- 黑盒 URL、Caido、浏览器、多目标和真实 PoC 都有实现，不是纯宣传。
- “SAST + DAST”中的 SAST 主要是 Agent 调 Semgrep/ast-grep；但这不影响黑盒评价。
- “自动修复”和部分企业能力不能全部外推到开源 CLI。

## 9. 评分

| 维度 | 分数 |
|---|---:|
| 资产/子域/API 发现 | 80 |
| 浏览器/代理/认证态 | 82 |
| 业务逻辑 | 68 |
| 自动利用与 PoC | 72 |
| 误报控制 | 70 |
| 证据与报告 | 80 |
| Scope/限速/审批 | 65 |
| 多目标与扩展 | 75 |
| 部署成本 | 55 |

**最终判断：70/100。** 若需要直接投入网站/API SRC，优先级应高于单看总分的排名。上线前应在容器网络层补域名/IP allowlist、重定向检查、速率和危险动作审批。

## 10. 关键源码依据

- `strix/core/inputs.py`
- `strix/core/agents.py`
- `strix/core/runner.py`
- `strix/tools/proxy/tools.py`
- `strix/tools/proxy/caido_api.py`
- `strix/tools/reporting/tool.py`
- `strix/report/writer.py`
- `strix/report/sarif.py`
- `strix/skills/reconnaissance/asset_discovery.md`
- `strix/skills/scan_modes/deep.md`
