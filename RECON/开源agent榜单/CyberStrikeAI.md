# CyberStrikeAI 黑盒 SRC 分析报告

> 评估场景：只有网站、API、域名、IP、测试账号和授权范围，没有目标源码。
>
> **综合评分：72/100。定位：平台型黑盒渗透中枢。**

## 1. 结论

CyberStrikeAI 是十个项目中平台治理能力最完整的一个：有多 Agent 编排、约 90 个 YAML 工具、MCP 联邦、项目事实黑板、结构化漏洞台账、RBAC、HITL、限速和审计。它适合团队化管理多项目渗透，也适合把 Burp、外部扫描器和自定义 MCP 汇入同一工作台。

但它不是开箱即用的认证态 Web Agent。核心工具集中没有完整原生浏览器，复杂登录、MFA、多角色切换和 DOM 交互主要依赖 Burp 插件、浏览器扩展或外部 MCP。若这些外围组件未接好，业务逻辑测试能力会明显下降。

## 2. 主要架构

- Go 后端 + Web UI，入口为 `cmd/server/main.go`。
- `internal/workflow/engine.go`、`eino_compile.go` 负责 Eino 工作流编排。
- 支持 Deep、Plan-Execute、Supervisor 和可视化图工作流。
- `agents/*.md` 定义侦察、渗透、分诊、报告等专员。
- `skills/*/SKILL.md` 提供攻击面、漏洞验证、源码狩猎等方法论。
- `tools/*.yaml` 和 MCP 联邦负责真实工具调用。
- SQLite 保存会话、任务、漏洞、项目事实、工具执行和审批记录。
- Burp 插件位于 `plugins/burp-suite/`，浏览器扩展位于 `plugins/browser-extension/`。

## 3. 黑盒能力

### 3.1 资产与 API 发现

侦察方法论覆盖域名、子域、端口、服务、目录、API、历史 URL 和组件情报。工具层可调用 nmap、nuclei、sqlmap、ffuf 等，也能通过 MCP 扩展 naabu/httpx/dnsx 等能力。

需要注意：Skill 中出现的部分工具名没有对应 YAML 定义。它们是 Agent 被告知可以使用的能力，不一定已经随仓库开箱安装。

### 3.2 浏览器、代理与认证态

- 有 Burp Suite 插件，可把请求送入 Repeater 并连接 Agent 工作流。
- 有浏览器扩展，可辅助采集当前浏览器上下文。
- 核心 `tools/` 中缺少完整的 `browser_navigate` 一类浏览器工具。
- 登录态、多角色和 MFA 的稳定性取决于插件或外部 MCP，而不是核心运行时保证。

### 3.3 业务逻辑与漏洞验证

LLM Agent 可按功能、角色和攻击链拆分任务，适合授权、IDOR、流程滥用和组合漏洞推理。`skills/pentest-verification/SKILL.md` 要求复现和确认影响。

实际验证仍依赖外部工具输出和模型判断，没有独立确定性的业务状态机或漏洞 oracle。

## 4. 证据与报告

`internal/app/vulnerability_tools.go` 的 `record_vulnerability` 强制填写：

- target；
- vulnerability type；
- description；
- reproduction steps；
- evidence/PoC；
- impact；
- recommendation。

项目 Fact 与漏洞记录分离：Fact 保存失败尝试、认证依赖、入口和环境事实，漏洞台账保存可交付 finding。这一设计适合长会话和跨 Agent 交接。

不足是 evidence 仍为自由文本，没有强制绑定浏览器截图、HTTP exchange ID 或文件化原始请求响应。

## 5. Scope、限速与运行安全

- 支持工具 allowlist、RBAC、审批模式、HITL 和审计。
- 有 Tool Guard 和速率限制实现，强于多数同类项目。
- Tool Guard 主要检查工具调用中可见的文本，不能可靠解析重定向、IP 归属、DNS 变化或工具从文件读取的间接目标。
- C2、WebShell 和后渗透模块对 SRC 通常多余，应默认关闭。

## 6. 部署成本

部署涉及 Go 服务、Web UI、数据库、LLM、YAML 工具和可选 MCP/插件。工具数量多，但大量外部二进制仍需单独安装配置。适合长期平台化运营，不适合临时单人快速起一个轻量扫描任务。

## 7. README 与实现差异

- “100+ 工具”总体属实，但不等于所有 Skill 引用的工具都已集成。
- “浏览器能力”主要由插件和外部 MCP 提供，核心 Agent 没有完整浏览器自动化。
- “源码狩猎/零日引擎”主要是方法论 Skill，不是内建程序分析引擎；不过这对黑盒 SRC 不是主要扣分项。

## 8. 评分

| 维度 | 分数 |
|---|---:|
| 资产/子域/API 发现 | 90 |
| 浏览器/代理/认证态 | 50 |
| 业务逻辑 | 75 |
| 自动利用与 PoC | 80 |
| 误报控制 | 70 |
| 证据与报告 | 85 |
| Scope/限速/审批 | 85 |
| 多目标与扩展 | 70 |
| 部署成本 | 60 |

**最终判断：72/100。** 最适合做团队化黑盒 SRC 平台中枢。投入实战前应补真实浏览器、稳定代理证据链和不可绕过的网络 egress scope。

## 9. 关键源码依据

- `cmd/server/main.go`
- `internal/workflow/engine.go`
- `internal/workflow/eino_compile.go`
- `internal/app/vulnerability_tools.go`
- `internal/security/executor.go`
- `tools/*.yaml`
- `skills/attack-surface-recon/SKILL.md`
- `skills/pentest-verification/SKILL.md`
- `plugins/burp-suite/`
- `plugins/browser-extension/`
