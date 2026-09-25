# PentAGI 黑盒 SRC 分析报告

> 评估场景：只有网站、API、域名、IP、测试账号和授权范围，没有目标源码。
>
> **综合评分：48/100。定位：基础设施成熟的通用多 Agent 渗透平台。**

## 1. 结论

PentAGI 的强项是多 Agent、Docker 沙箱、长期记忆、任务分解、真实 CLI 工具和可观测性，适合长时间、多目标的通用渗透任务。它能让 Agent 调用 nmap、sqlmap、Metasploit，生成 PoC，并把经验存入 pgvector。

但网站 SRC 最关键的认证态浏览器、HTTP 拦截重放、硬性 scope、速率控制和结构化漏洞证据都不够。`browser` 名称容易误导：实现主要是 `net/http` 爬取器，不是执行 JS 的真实浏览器。

## 2. 主要架构

- Go 后端 + React 前端，入口 `backend/cmd/pentagi/main.go`。
- Flow → Task → SubTask → Action/Artifact 的任务层次。
- primary agent 委派 pentester、coder、searcher、installer、memorist 和 reporter。
- `backend/pkg/tools/registry.go` 注册 terminal、browser、code、file diff 和搜索工具。
- Docker 沙箱执行真实安全工具。
- PostgreSQL/pgvector 存长期记忆，可选 Graphiti/Neo4j。
- Langfuse/OpenTelemetry 提供观测。

## 3. 黑盒能力

### 3.1 资产与工具执行

Pentester 可在 Kali 沙箱中运行 nmap、sqlmap、Metasploit 和其他 CLI。Searcher 负责网络情报，Coder 可生成辅助脚本和 PoC。多 Agent 任务结构适合把多个目标拆成并行子任务。

它没有内建统一资产图、子域流水线或 API inventory，实际覆盖依赖模型是否正确选择工具和保存结果。

### 3.2 浏览器与认证态

`browser.go` 使用 `net/http` 获取页面，适合只读抓取，但存在决定性限制：

- 不执行 JavaScript；
- 不能可靠点击和填写表单；
- 没有成熟的 Cookie/多角色会话管理；
- 不提供 Burp/Caido 式拦截和重放；
- 截图依赖外部服务。

因此它不适合复杂登录、MFA、SPA、IDOR 和多步骤业务流程。

## 4. 业务逻辑、PoC 与误报

LLM 可对流程和授权逻辑做推理，Coder 能写 exploit/PoC，Terminal 能验证影响。planner、mentor、reflector 和 adviser 有助于纠正执行偏差。

但项目没有独立 retester、反证面板或统一的“无证据不确认”状态机。报告和 memory 中的结果仍可能是 Agent 总结，不能自动等同于已复现漏洞。

## 5. 证据与报告

- reporter 能汇总 task/subtask/action 结果；
- `hack_result`、`report_result` 和 artifact 可落数据库；
- `store_code/search_code` 可沉淀可复用 PoC 片段。

不足：没有强制漏洞 schema，不要求绑定原始请求响应、截图、复现步骤和影响；code 工具是代码/利用片段库，不是目标漏洞数据模型。

## 6. Scope、限速与安全性

文档和 prompt 建议明确 allowed targets、out-of-scope 和 stop conditions，但主要依靠 Agent 遵守。项目没有统一的目标 allowlist、DNS/IP 解析、重定向检查和网络 egress hard gate，也没有针对真实 SRC 的全局速率策略。

Docker 降低了工具对宿主机的影响，但不能防止容器向越界网络目标发请求。

## 7. 部署成本

需要 Go/React 服务、PostgreSQL/pgvector、Docker 和 LLM；可选 Neo4j、Graphiti、Langfuse 和监控栈。部署和资源成本较高，适合平台化使用，不适合轻量一次性 SRC。

## 8. README 与实现差异

README 的多 Agent、Docker、记忆和真实工具基本属实。容易误解的两点：

- browser 是 HTTP 抓取器，不是浏览器自动化；
- code 工具是 PoC/代码片段库，不是源代码或漏洞分析引擎。

## 9. 评分

| 维度 | 分数 |
|---|---:|
| 资产/子域/API 发现 | 55 |
| 浏览器/代理/认证态 | 30 |
| 业务逻辑 | 45 |
| 自动利用与 PoC | 60 |
| 误报控制 | 45 |
| 证据与报告 | 50 |
| Scope/限速/审批 | 40 |
| 多目标与扩展 | 80 |
| 部署成本 | 35 |

**最终判断：48/100。** 适合复用为多 Agent 执行和记忆平台；用于网站 SRC 前必须增加真实浏览器、HTTP 代理、scope 网关、速率控制和强制 finding schema。

## 10. 关键源码依据

- `backend/cmd/pentagi/main.go`
- `backend/pkg/controller/flow.go`
- `backend/pkg/controller/subtask.go`
- `backend/pkg/tools/registry.go`
- `backend/pkg/tools/terminal.go`
- `backend/pkg/tools/browser.go`
- `backend/pkg/tools/code.go`
- `backend/pkg/templates/prompts/pentester.tmpl`
- `backend/pkg/templates/prompts/coder.tmpl`
- `examples/prompts/scope_of_work_pentest.md`
