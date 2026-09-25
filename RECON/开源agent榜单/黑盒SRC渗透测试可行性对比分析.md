# 开源安全 Agent：黑盒 SRC 渗透测试可行性对比分析

> 分析对象：`Security/Offensive` 下 10 个本地项目快照。
>
> 本文中的 **SRC** 指企业安全响应中心、漏洞赏金或众测场景。典型输入只有网站、API、域名、IP、测试账号和范围说明，**通常没有目标源码**。评估目标是：能否在授权范围内完成黑盒侦察、漏洞验证和可提交报告，而不是源码审计。

## 1. 结论摘要

### 1.1 综合排名

| 排名 | 项目 | 黑盒 SRC 分 | 定位 | 核心判断 |
|---:|---|---:|---|---|
| 1 | CyberStrikeAI | 72/100 | 平台型 | 工具编排、项目黑板、漏洞台账、审批和审计最完整；原生认证态浏览器不足 |
| 2 | T3MP3ST | 71/100 | 研究型 | 硬性 scope、风险审批、证据门和反误报最强；MITM 与认证态较弱，部分能力仍实验性 |
| 3 | Strix | 70/100 | 实战型 | Caido 代理、浏览器、API、认证态和报告链最适合直接做 Web SRC；scope 主要依赖 Agent 自守 |
| 4 | HexStrike AI | 50/100 | 工具网关 | 资产和 API 工具覆盖很广，但没有真实 LLM 推理，缺 scope/审批，验证与报告较弱 |
| 5 | PentAGI | 48/100 | 通用平台 | Docker、多 Agent、记忆和真实 CLI 工具扎实；浏览器只是 HTTP 爬虫，缺硬性范围控制 |
| 6 | CAI | 45/100 | Agent 框架 | 有专用 bug bounty/retester/reporter 链和业务逻辑推理；无浏览器/代理/认证态，且已归档 |
| 7 | OWASP Nettacker | 40/100 | 确定性扫描器 | 侦察、已知 CVE、弱口令、事件落库和多格式报告稳定；不能发现业务逻辑和未知漏洞 |
| 8 | PentestAgent | 36/100 | 轻量 Agent | 有终端、Playwright 和多 Agent，但报告、证据、误报控制和 scope 护栏不足 |
| 9 | PentestGPT | 35/100 | 执行控制器 | Claude Code/Codex 加精确命令证据有研究价值；无浏览器/代理，finding 与正式报告未实现 |
| 10 | Shannon | 35/100 | 灰盒工具 | 动态利用和报告很强，但 CLI 强制同时提供源码仓库，典型无源码 SRC 无法启动 |

分数接近不代表产品可互换。前三名分别偏向平台治理、安全护栏和 Web 实战闭环。

### 1.2 直接选型建议

- **认证态 Web/API、Burp/Caido 式流量重放、直接产出可提交报告：优先 Strix。** 它不是总分最高，但最贴近日常网站 SRC 操作。
- **强制 scope、危险操作审批、证据溯源、降低越界和误报风险：优先 T3MP3ST。** 适合作为受控执行框架，但认证态测试要补代理和浏览器能力。
- **需要多人、多项目、工具治理、RBAC、知识库和漏洞生命周期：优先 CyberStrikeAI。** 需要接入真实浏览器或 Burp MCP 才能补齐复杂登录态。
- **只做大范围资产发现和已知漏洞初筛：HexStrike AI 或 Nettacker 可作前置层。** 不能把扫描器命中直接当作可提交漏洞。
- **Shannon 不适合无源码 SRC。** 它适合“源码 + 可运行站点”的灰盒/白盒验证，不应与纯黑盒项目直接比较。

## 2. 评估口径

### 2.1 黑盒 SRC 的完整链路

1. 接收域名、IP、URL、API 描述、测试账号及 in-scope/out-of-scope 规则；
2. 被动与主动发现子域、端口、服务、目录、参数、API 和历史 URL；
3. 通过浏览器或代理维持登录态，区分用户角色并重放、修改请求；
4. 测试认证、授权、IDOR、业务状态机、竞态、注入、SSRF、XSS、文件处理等；
5. 对候选漏洞做复测、反证、去重和最小影响 PoC；
6. 保存原始请求/响应、命令输出、截图、前置条件和影响证明；
7. 输出可提交给 SRC 的复现步骤、影响、严重性和修复建议；
8. 全程执行范围、速率、并发、危险动作审批和数据最小化约束。

### 2.2 评分维度

| 维度 | 权重 | 关注点 |
|---|---:|---|
| 资产、子域和 API 发现 | 15 | 子域、端口、目录、参数、JS/API、历史资产、技术指纹 |
| 浏览器、代理和认证态 | 15 | JS 浏览器、Cookie/会话、MFA、多角色、拦截与重放 |
| 业务逻辑能力 | 15 | 授权、IDOR、状态机、竞态、支付/优惠、跨步骤攻击 |
| 自动利用和 PoC | 15 | 真实工具、脚本、可重复利用和影响证明 |
| 误报控制 | 10 | 复测、反证、去重、置信度和“无证据不确认” |
| 证据和 SRC 报告 | 10 | 原始流量、截图、命令回执、结构化 finding、报告格式 |
| Scope、限速和审批 | 10 | 硬边界、出站控制、速率、并发、危险操作 HITL |
| 多目标、恢复和扩展 | 5 | 并行、断点恢复、多目标、工具扩展、长期状态 |
| 部署与维护成本 | 5 | 安装复杂度、外部依赖、维护状态和运行安全性 |

## 3. 横向能力矩阵

| 项目 | 无源码 URL/IP | 子域/API 发现 | 浏览器/代理 | 认证态 | 业务逻辑 | PoC/复测 | Scope 硬门 | SRC 报告 |
|---|---|---|---|---|---|---|---|---|
| CyberStrikeAI | 是 | 强，100+ YAML/MCP | Burp/浏览器插件，核心无原生浏览器工具 | 中，依赖插件/MCP | 强，LLM+技能 | 强，漏洞台账强制证据 | 中强，规则+审批，但间接访问有盲区 | 强，结构化漏洞库 |
| T3MP3ST | 是 | 强，Arsenal 工具链 | 弱，无 MITM；浏览器能力有限 | 弱到中 | 中 | 强，evidence gate+retest+refuter | 强，egress 硬门+风险审批 | 中，Markdown/披露包 |
| Strix | 是 | 强，URL/IP/OpenAPI/多目标 | 强，Caido+agent_browser | 强，可传凭据并保留流量 | 中强 | 强，PoC+counterevidence | 中，目标写入系统提示，缺 egress 硬门 | 强，JSON/CSV/MD/PDF/SARIF |
| HexStrike AI | 是 | 很强，150+ 外部工具入口 | Selenium，无成熟 MITM 证据链 | 弱到中 | 弱，规则模板 | 中，依赖外部工具 | 很弱 | 弱到中，主要是 API/终端输出 |
| PentAGI | 是 | 中强，真实 Kali 工具 | 弱，`browser.go` 是 HTTP 爬虫 | 弱 | 中，LLM 推理 | 中强，Docker 执行 | 弱，主要靠提示词 | 中，通用 reporter |
| CAI | 是 | 中强，Shodan/C99/shell | 弱，无真实浏览器和代理 | 弱 | 中强，专用 bug bounty Agent | 中，bug bounty↔retester swarm | 弱到中，主要靠提示词 | 中，Reporter 输出 HTML |
| Nettacker | 是 | 中，子域/端口/目录/服务 | 无 | 无 | 无 | 中，模板响应匹配 | 中，延时/线程参数但无审批 | 强，DB+HTML/JSON/CSV/SARIF |
| PentestAgent | 是 | 中，依赖 Agent 调 CLI | 中，本地 Playwright；Docker 路径部分为 curl stub | 中 | 中 | 中 | 弱，浅层参数检查有绕过面 | 弱到中，notes→Markdown |
| PentestGPT | 是，HTTP(S) target | 弱到中，依赖 coding CLI | 无原生浏览器/代理 | 弱 | 中，模型推理 | 中，精确 receipt evidence | 弱到中，目标状态机 | 弱，finding/report 未实现 |
| Shannon | 否，必须同时给 repo | 很弱，无资产测绘 | 强，Playwright+认证流程 | 强 | 中，仅固定 Web 类别 | 强，no exploit no report | 弱到中，ROE 主要由提示词执行 | 强，但黑盒场景无法启动 |

## 4. 逐项目分析

### 4.1 CyberStrikeAI

**架构与特点**

- Go 单体后端 + Web UI，基于 Eino，支持 Deep、Plan-Execute、Supervisor 和图工作流。
- `agents/*.md` 定义侦察、渗透、漏洞分诊和报告专员；`skills/` 提供方法；`tools/*.yaml` 与 MCP 联邦负责实际执行。
- SQLite 持久化会话、项目 Fact 黑板、漏洞、工具调用和审批；提供 RBAC、HITL、审计、知识库、攻击链与回放。
- 另有 Burp Suite 插件和 Chrome/Edge 扩展，但它们是外围集成，不等于核心 Agent 自带完整浏览器工具。

**黑盒 SRC 优势**

- 工具面覆盖侦察、Web、API、凭据、云、二进制和后渗透，适合把多种外部能力放进统一工作台。
- `internal/app/vulnerability_tools.go` 的 `record_vulnerability` 强制 target、类型、描述、复现步骤、evidence、impact 和 recommendation，明显优于自由文本 notes。
- Agent 被要求边测试边 `upsert_project_fact`，失败尝试、认证上下文和复现依赖不容易因上下文压缩丢失。
- 图工作流、审批模式、工具 allowlist、RBAC 和执行审计适合团队化 SRC。

**限制与风险**

- Skills 中出现的 `browser_navigate`、naabu/httpx/dnsx 等能力并非都在 `tools/` 有对应定义；部分属于“提示词知道有这个工具”，需要外部 MCP 或人工补齐。
- 无核心原生认证态浏览器。复杂 MFA、多角色切换、DOM 操作和请求重放依赖 Burp/浏览器插件的实际接入质量。
- Tool Guard 的界面说明承认规则只检查调用中可见文本，不能解析重定向、IP 归属或工具从文件读取的间接目标，因此不是绝对 egress 隔离。
- 部署和运维较重，C2/WebShell 对 SRC 多余，并增加合规和攻击面。

**判断：72/100。** 最适合做黑盒 SRC 的平台中枢；要形成稳定实战闭环，需补强浏览器/代理和硬性出站范围控制。

### 4.2 T3MP3ST

**架构与特点**

- TypeScript，包含 ReAct AgentLoop、MissionControl、8 类 operator、Arsenal 工具库、PackBoard、evidence/retest/reporting 和 Web War Room。
- 可复用 Claude Code/Codex 等本地登录 Agent，也支持 API 模型。
- Arsenal catalog 集成 nmap、subfinder、httpx、naabu、katana、nuclei、ffuf、amass、dalfox、sqlmap 等。

**黑盒 SRC 优势**

- `arsenal/index.ts` 在工具执行前做 egress scope 检查，越界直接拒绝，不只依赖系统提示。
- `arsenal/approval.ts` 按 intrusive/credential/dangerous 风险分级审批；无人审批时 fail-safe 拒绝高风险动作。
- `evidence/gate.ts` 区分工具证据和模型断言，无真实证据不能标记 verified。
- `evidence/retest.ts`、`mission/adjudicate.ts` 和 `verify-finding.mjs` 提供复测、反证、引用检查、PoC 阶梯与去重。
- 侦察工具面广，多目标和并行框架完整，XBOW 黑盒结果至少有可重算 verdict 材料。

**限制与风险**

- 没有 Caido/Burp 式 MITM 代理。`net/proxy.ts` 主要是 SOCKS5 出站代理，并非 HTTP 拦截与流量证据系统。
- 浏览器路径偏 GET/XSS 探针，复杂登录、MFA、多角色权限和表单工作流明显弱于 Strix。
- README 明确后续利用 operator 和 swarm 仍属 experimental；被 benchmark 的主要是单 Agent ReAct 路径。
- 自动报告以 Markdown/披露包为主，没有成熟 PDF/SARIF 管线，部分文案本地化不适合直接提交。

**判断：71/100。** 合规护栏和反误报设计最值得复用；补齐认证态浏览器、MITM 和报告模板后，才适合成为主力 Web SRC 工具。

### 4.3 Strix

**架构与特点**

- Python `openai-agents` + LiteLLM，`AgentCoordinator` 管理可寻址 Agent 图、mailbox、状态、预算和快照。
- Docker 沙箱提供真实安全工具，Caido 代理记录/重放 HTTP，agent_browser 负责浏览器交互。
- 输入支持 URL、IP、本地目录、GitHub、OpenAPI/Postman 和多目标列表。

**黑盒 SRC 优势**

- `skills/reconnaissance/asset_discovery.md` 覆盖 CT、被动 DNS、ASN、subfinder、httpx、naabu；同时有 katana、ffuf、nuclei、sqlmap 等。
- Caido 项目保存请求响应，reporting 工具会校验 `http_exchange_ids`；PoC、evidence、counterevidence、confidence 和 CVSS 都进入结构化 finding。
- 浏览器 + 代理适合维持 Cookie、测试多角色、修改请求、验证 IDOR/认证绕过和记录可提交证据。
- 输出逐 finding Markdown、JSON、CSV、PDF 和 SARIF；`--target-list` 与多 Agent 支持批量目标。

**限制与风险**

- `authorized_targets` 被注入系统提示并声明不可扩展，但没有像 T3MP3ST 那样在每次出站执行前做强制 egress 判定。
- 限速和非破坏性操作主要依赖 skill/提示词，缺统一风险审批门。
- 覆盖率和测试完成度仍受 Agent 自报、模型质量和 token 预算影响。
- SARIF 中的源码位置只在有源码时有意义；纯黑盒应以 HTTP exchange、PoC 和截图为主要证据。

**判断：70/100。** 如果目标是直接做网站/API SRC，它是当前十项中最实用的一体化选择；总分略低主要因为硬性 scope/审批不足。

### 4.4 HexStrike AI

**架构与特点**

- `hexstrike_mcp.py` 暴露大量 MCP 工具，转发给 Flask `hexstrike_server.py`，服务端以 subprocess 调外部 CLI。
- 覆盖 amass、subfinder、httpx、katana、gau、waybackurls、arjun、paramspider、nuclei、sqlmap、Dalfox、GraphQL/JWT/API scanner 和 Selenium。

**黑盒 SRC 优势**

- 资产、参数、API 和已知漏洞工具入口非常广，适合让外部 Claude/Cursor 作为前端统一调用。
- 外部工具是真实执行，不是全部模拟；侦察和批量初筛价值较高。

**限制与风险**

- 项目内部没有真实 LLM；所谓 decision engine、payload generator 和 autonomous agents 主要是打分表、关键词和字符串模板。
- Flask 默认监听 `0.0.0.0:8888`，缺成熟鉴权；部分命令通过 `shell=True` 拼接，运行安全边界差。
- 无可靠 scope、限速和危险动作审批。150+ 工具还需另行安装，README 的“开箱”程度被高估。
- 风险评分和漏洞判断大量依赖关键词，缺复测、反证和结构化 SRC 报告；README 中检测率/成功数没有可复核测量实现。

**判断：50/100。** 可作为前置侦察和 MCP 工具网关，不适合在没有外部治理层时自主跑真实 SRC。

### 4.5 PentAGI

**架构与特点**

- Go 后端 + React 前端；primary agent 将 Flow 拆成 Task/SubTask，委派 pentester、coder、searcher、installer 和 reporter。
- Docker 沙箱执行 nmap、sqlmap、Metasploit 等工具；PostgreSQL/pgvector 记忆，可选 Graphiti/Neo4j 和完整观测栈。

**黑盒 SRC 优势**

- 多 Agent 任务分解、Docker 执行、长期记忆、摘要和监督机制成熟，适合长时间多阶段渗透。
- Terminal 是真实工具执行，coder 可生成 PoC，reporter 可汇总结果。

**限制与风险**

- `browser.go` 本质是 `net/http` 抓取器，不执行 JS、不能可靠操作表单、保持复杂会话或截图；README 的 browser 容易让人误认为真实浏览器自动化。
- 缺强制目标白名单、出站 scope 和统一速率控制，主要依赖任务描述和提示词。
- 漏洞结果不是强 schema，证据、原始请求响应和复测字段不如 CyberStrikeAI/Strix。
- 部署依赖 PostgreSQL/pgvector、Docker 和多种可选服务，资源和 token 成本较高。

**判断：48/100。** 通用自主渗透底座不错，但要用于网站 SRC，必须补真实浏览器、代理、范围硬门和 finding 模型。

### 4.6 CAI

**架构与特点**

- Python 多 Agent 框架，包含 bug_bounter、web_pentester、retester、reporter 和多种 orchestration/handoff pattern。
- `bug_bounter.py` 实际工具为通用 shell、代码执行、Shodan、Web 情报和可选 C99/Google；`bb_triage.py` 在发现 Agent 与复测 Agent 间循环 handoff。

**黑盒 SRC 优势**

- `system_bug_bounter.md` 明确覆盖范围定义、子域/API、角色、认证授权、业务逻辑、竞态、PoC 和负责任披露。
- 专门的 retester Agent 能降低一部分误报，Reporter 不带执行工具，只根据会话生成 HTML，减少报告阶段继续执行攻击的风险。
- LLM + 通用 shell 灵活，能临时调用 ffuf/sqlmap/nuclei 等系统工具。

**限制与风险**

- 无真实浏览器、MITM 代理和稳定 Cookie 会话；`tools/web` 主要是 fetch/headers/search，不适合复杂认证态。
- ffuf/sqlmap 等多数并非封装工具，而是由 LLM 在 shell 手工调用；环境缺工具时能力直接消失。
- scope、限速和危险动作审批主要是提示词规则；guardrail 更关注 prompt injection，不是网络出站硬门。
- 证据主要存在于会话文本，缺原始请求存档和强制 finding schema；项目已归档，且默认模型/部分能力依赖外部服务。

**判断：45/100。** 适合作为业务逻辑推理和复测 Agent 的参考实现，不适合单独承担认证态 Web SRC 全流程。

### 4.7 OWASP Nettacker

**架构与特点**

- Python CLI + Flask API/Web UI；YAML 模块定义请求、payload 和响应条件。
- 多进程按目标、线程按模块并发；事件写入数据库，支持历史扫描对比。

**黑盒 SRC 优势**

- 适合子域、端口、服务、目录、默认凭据和已知 CVE 的确定性前置扫描。
- 完整请求/响应事件可落库，并输出 HTML、JSON、CSV、SARIF 和 DefectDojo 格式。
- `time_sleep`、重试、代理和 `thread_per_host` 提供基础负载控制；多目标规模化成熟。

**限制与风险**

- 无浏览器、代理重放、登录态、多角色测试或业务逻辑推理。
- 漏洞判断是 YAML 正则/条件签名，主要发现已知 CVE 和配置问题，不能自主发现未知漏洞。
- 所谓 drift detection 主要比较目标、模块和端口，不是语义级漏洞生命周期管理。

**判断：40/100。** 适合作为确定性侦察/已知漏洞扫描层，不适合作为独立 SRC 猎洞 Agent。

### 4.8 PentestAgent

**架构与特点**

- Python TUI/CLI/MCP，LiteLLM；单 Agent plan/finish 与 Crew orchestrator/WorkerPool 双模式。
- 终端、Playwright 浏览器、notes、RAG、ShadowGraph 和可自孵化 MCP 子 Agent。

**黑盒 SRC 优势**

- 本地 runtime 有真实 Playwright，终端可调用常规渗透工具；整体轻量，易接入自有 MCP。
- Workspace 能维护目标列表，并有基础目标规范化与 scope 检查。

**限制与风险**

- Docker runtime 的 browser 路径仍是 curl/grep stub，截图不可用；不同 runtime 能力不一致。
- `gather_candidate_targets` 只检查顶层常见参数，不递归解析命令、重定向和嵌套参数，无法视为可靠出站边界。
- 无强制 PoC/反证/finding schema；主要把自由文本 notes 生成 Markdown，证据和误报控制较弱。
- 无内建资产发现引擎，扫描质量取决于模型是否正确选择并解析外部 CLI。

**判断：36/100。** 可作为轻量实验框架；没有额外 scope 网关、代理证据和报告层时，不建议自主跑生产 SRC。

### 4.9 PentestGPT

**架构与特点**

- Python 小型双角色循环：Supervisor 每次选择一个任务，Executor 通过 Claude Code/Codex FULL_ACCESS 执行。
- SQLite 保存任务、lease、attempt、精确 observation 和 revision，支持恢复和审计。

**黑盒 SRC 优势**

- 命令回执证据要求严格：观察必须引用真实 action receipt 的精确连续片段，传输错误不能当证据。
- coding agent 的通用 shell 能临时完成侦察、curl 测试和 PoC 编写。

**限制与风险**

- 无原生浏览器、MITM、认证态、子域/API 发现和专用 Web 工具编排。
- Structured finding、正式报告和 UI 在 README 中明确仍未实现；现有完成状态不等于漏洞结论正确。
- 流程偏 CTF/flag oracle，单任务串行，历史实战记录显示控制器重复 discovery 和未完成目标。
- FULL_ACCESS 依赖外围容器/VM 隔离，不适合直接运行在敏感主机。

**判断：35/100。** 证据状态机值得借鉴，但当前不是可直接提交 SRC 漏洞的完整产品。

### 4.10 Shannon

**架构与特点**

- TypeScript + Temporal，CLI 启动 Worker/Docker，把源码只读挂载后执行代码感知侦察、固定漏洞类别分析、利用和报告。
- Playwright 支持登录流程、TOTP、邮箱 OTP/SSO；只有真实 exploit 的 finding 才进入最终报告。

**黑盒 SRC 优势**

- 在同时拥有源码和测试站点时，认证态、PoC 验证、误报控制和 PDF/MD/JSON/SARIF 报告都很强。

**决定性限制**

- `apps/cli/src/index.ts` 明确报错 `--url and --repo are required`，`start.ts` 无条件解析 repo；Worker 工作流也依赖 repoPath。
- 因而它没有“只给 URL/IP”的黑盒模式。无源码时不是效果下降，而是产品入口不成立。
- 无子域/ASN/大范围资产发现，主要从源码关联单应用攻击面；规则范围集中在注入、XSS、SSRF、认证和授权。
- `rules_of_engagement` 主要由提示词执行，缺统一 egress scope 和危险动作审批门。

**判断：35/100。** 这是条件性能力分；在典型无源码 SRC 中应直接排除。若项目方同时提供源码，它可跃升为强灰盒方案，但那是另一种场景。

## 5. 关键纠偏

| 容易产生的误判 | 源码事实 | 正确结论 |
|---|---|---|
| SRC 被理解为 Source Code | 这些项目多数以 URL/IP/域名为输入 | SRC 应按安全响应中心/漏洞赏金的黑盒网站测试评估 |
| Shannon 是黑盒自动渗透第一梯队 | CLI 强制 `--url` 与 `--repo` | 无源码 SRC 无法使用；它是灰盒/白盒工具 |
| 工具数量多等于猎洞能力强 | HexStrike 有 150+ 接口但内部无 LLM，很多决策是模板 | 适合侦察网关，不等于能理解业务逻辑或控制误报 |
| 有 browser 文件就有浏览器 | PentAGI 是 HTTP 爬虫；PentestAgent Docker 路径部分为 curl stub | 必须核对 JS、Cookie、表单、截图、代理和请求重放实现 |
| Prompt 中写 scope 就能防越界 | 多数项目由 LLM 自守，间接目标和重定向可绕过 | T3MP3ST 的执行前 egress gate 更接近真实硬边界 |
| Nuclei/CVE 命中即可提交 | 模板命中可能是版本/响应特征，不一定有影响证明 | 必须增加复测、PoC、原始流量和业务影响 |
| Benchmark 证明完整产品能力 | T3MP3ST benchmark 是单 Agent；PentestGPT runner 不在产品仓库 | 只能证明特定 harness/模型/目标，不能外推全链路 |

## 6. 推荐落地架构

单个项目都没有完整覆盖“资产 → 认证态 → 业务逻辑 → PoC → 证据 → 合规”的全部环节。更现实的组合是：

1. **资产层**：subfinder/amass/httpx/naabu/katana/gau/waybackurls/ffuf，结合 Nettacker 或 HexStrike 统一调度；
2. **交互层**：Strix 的 Caido + browser，或独立 Burp/Playwright 服务，保存所有关键 HTTP exchange；
3. **推理层**：Strix/CyberStrikeAI/CAI 风格的专员 Agent，按资产、角色、功能和漏洞类拆任务；
4. **安全层**：采用 T3MP3ST 式 egress scope 硬门、风险分级审批、速率和并发策略；
5. **验证层**：候选必须经独立 retester，保存 counterevidence，禁止仅凭扫描器标题确认；
6. **状态层**：CyberStrikeAI 式 Fact 黑板与 finding 分离，记录失败尝试、认证依赖和漏洞生命周期；
7. **报告层**：强制 target、前置条件、复现步骤、原始请求响应、PoC、影响、严重性、修复和复测状态。

### 建议的实际试点

选择 3 类授权目标，用固定模型和预算对 Strix、T3MP3ST、CyberStrikeAI 做同场测试：

- 多角色 SaaS：重点测 IDOR、租户隔离、审批流和业务状态机；
- API 平台：重点测 OpenAPI 覆盖、鉴权、对象级授权、批量接口和速率限制；
- 多子域互联网资产：重点测资产召回、历史 URL、接管、配置和已知组件漏洞。

记录资产召回率、有效漏洞数、人工确认 precision、PoC 可复现率、越界请求数、请求负载、单 finding 成本和重复运行稳定性。没有统一实测前，本文评分应视为**基于项目实现的工程可行性评分**，不是漏洞发现率排名。

## 7. 本地源码快照

| 项目 | Commit | Commit 日期 |
|---|---|---|
| Shannon | `327c10f` | 2026-09-21 |
| Strix | `ae38fe7` | 2026-09-23 |
| T3MP3ST | `29824d5` | 2026-09-07 |
| CyberStrikeAI | `e9b6e0d` | 2026-09-23 |
| PentestAgent | `87d763a` | 2026-09-21 |
| PentAGI | `ea66530` | 2026-08-06 |
| CAI | `6dc7925` | 2026-08-22 |
| PentestGPT | `e8b1bb7` | 2026-07-14 |
| HexStrike AI | `d689933` | 2026-08-03 |
| OWASP Nettacker | `cee419f` | 2026-09-22 |

日期取自本地 Git 元数据，只用于标识分析快照，不代表正式发布日期。

## 8. 分析边界

- 本报告基于本地源码、配置、提示词和文档交叉核对，未部署十项工具对同一真实 SRC 目标做统一 benchmark。
- 评分反映架构与工程实现对黑盒 SRC 的适配度，不等于真实漏洞召回率。
- 实际效果强依赖模型、工具安装、账号质量、目标技术栈、WAF、预算和程序规则。
- 所有主动测试都必须在书面授权范围和 SRC 规则允许的方式、时间、速率内进行。
