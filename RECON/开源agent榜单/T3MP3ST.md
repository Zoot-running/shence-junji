# T3MP3ST 黑盒 SRC 分析报告

> 评估场景：只有网站、API、域名、IP、测试账号和授权范围，没有目标源码。
>
> **综合评分：71/100。定位：强护栏、强证据的研究型黑盒 Agent。**

## 1. 结论

T3MP3ST 最突出的不是工具数量，而是执行前 scope 硬门、危险动作审批、证据晋级、独立复测和反证机制。在真实 SRC 中，这些能力直接决定是否越界、是否把模型猜测误报成漏洞。

它的主要短板同样明确：没有 Burp/Caido 式 MITM 代理，浏览器偏轻量 GET/XSS 探测，复杂登录、MFA、多角色和请求重放能力不足。它更适合做受控侦察、工具编排和漏洞验证框架，而不是直接替代人工 Burp 工作流。

## 2. 主要架构

- TypeScript/Node.js，提供 CLI、HTTP API、War Room 和 MCP。
- `src/agent/index.ts` 实现 ReAct AgentLoop。
- MissionControl/TaskQueue 调度 8 类 operator。
- `src/arsenal/` 管理工具 catalog、参数、scope 和审批。
- PackBoard 用于共享线索、认领和去重。
- `src/evidence/`、`src/mission/adjudicate.ts` 和 scripts 负责证据、复测和披露。
- 支持 Claude Code、Codex 等本地登录 Agent，也支持 API 模型。

## 3. 黑盒能力

### 3.1 资产与 API 发现

Arsenal catalog 覆盖 nmap、subfinder、httpx、naabu、katana、nuclei、ffuf、amass、dalfox、sqlmap 等。另有 attack graph、参数拆分、子域接管和 IDOR 辅助模块。

这套工具面适合从域名扩展到子域、端口、URL、参数和已知漏洞，也支持多目标与并行任务。

### 3.2 浏览器、代理与认证态

- `arsenal/browser.ts` 能完成有限的网页探测和 XSS 相关操作。
- 不具备成熟的 JS 浏览器会话、复杂表单、MFA 和多角色切换能力。
- `net/proxy.ts` 是 SOCKS5 出站代理，用于源 IP、地理和 WAF 场景，不是 HTTP MITM 代理。
- 无原生请求历史、拦截、修改、重放和 exchange ID 证据系统。

因此对认证后 IDOR、流程绕过、支付/审批等业务逻辑，仍需要外接 Burp/Playwright。

## 4. 误报控制和证据

这是 T3MP3ST 的最强部分：

- `evidence/gate.ts` 要求 finding 有真实工具输出才能标记 verified；
- 明确区分 tool evidence 与 model assertion；
- `evidence/retest.ts` 执行复测，并要求显式 scope；
- `mission/adjudicate.ts` 提供 refuter 反证和引用检查；
- `scripts/verify-finding.mjs` 检查 anchors、PoC 层级和 novelty；
- PackBoard 支持线索背书、驳斥和去重。

报告以 Markdown 和披露包为主，没有 Strix 那样成熟的 PDF、代理流量和 SARIF 输出。

## 5. Scope、审批与安全性

- Arsenal 每次执行前做 egress scope 检查，越界直接 `SCOPE DENIED`。
- `arsenal/approval.ts` 按 intrusive、credential、dangerous 风险等级要求审批。
- 无审批者时高风险能力 fail-safe 拒绝，而不是默认放行。
- 本地文件也有 `T3MP3ST_SOURCE_ROOT` 路径围栏。

这是十个项目中最接近“执行层强制边界”的实现。限速仍需根据目标和工具进一步配置。

## 6. 成熟度和部署

- npm 安装，可使用现有本地 coding Agent，模型接入灵活。
- Recon 被标为稳定，后续利用 operator 和 swarm 仍为 experimental。
- README 的高分 benchmark 来自单 Agent ReAct，不是 8 operator 蜂群。
- `verify-claims` 能重算已提交 verdict 和统计，但原始逐步轨迹不完整，不能等同独立复跑。

## 7. README 与实现差异

README 对状态相对诚实：稳定、实验性和 stub 有明确标记。需要避免把“框架中存在 operator”理解为“该 operator 已经过端到端验证”，也不能把单 Agent benchmark 外推为 swarm 成绩。

## 8. 评分

| 维度 | 分数 |
|---|---:|
| 资产/子域/API 发现 | 85 |
| 浏览器/代理/认证态 | 58 |
| 业务逻辑 | 60 |
| 自动利用与 PoC | 70 |
| 误报控制 | 82 |
| 证据与报告 | 68 |
| Scope/限速/审批 | 85 |
| 多目标与扩展 | 70 |
| 部署成本 | 60 |

**最终判断：71/100。** 适合重视授权边界、危险动作审批和低误报的 SRC。要成为主力 Web 工具，需要补认证态浏览器、MITM 代理和标准化中文/英文报告。

## 9. 关键源码依据

- `src/agent/index.ts`
- `src/arsenal/index.ts`
- `src/arsenal/catalog.ts`
- `src/arsenal/approval.ts`
- `src/arsenal/browser.ts`
- `src/net/proxy.ts`
- `src/evidence/gate.ts`
- `src/evidence/retest.ts`
- `src/mission/adjudicate.ts`
- `scripts/verify-finding.mjs`
