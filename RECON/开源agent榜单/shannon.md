# Shannon 黑盒 SRC 分析报告

> 评估场景：只有网站、API、域名、IP、测试账号和授权范围，没有目标源码。
>
> **综合评分：35/100。定位：强灰盒 Web 验证工具，但不支持典型无源码 SRC。**

## 1. 结论

Shannon 的动态利用、认证流程、误报控制和报告都很强，但它在产品入口上强制同时要求运行 URL 和源码仓库。对于只有网站/API 的常规 SRC，这不是“效果下降”，而是无法按正常流程启动。

因此 Shannon 在本榜单中排末尾，不代表它本身质量差，而是场景不匹配。若厂商同时提供源码和 staging，它会变成很有竞争力的灰盒方案。

## 2. 主要架构

- TypeScript monorepo，CLI 启动本地 Docker/Worker。
- Temporal 编排确定性工作流，Activity 承担模型调用和外部副作用。
- 源码仓库只读挂载后，先做代码感知 recon 和漏洞分析，再做 reconciliation、exploit 和报告。
- 固定覆盖 injection、XSS、SSRF、authentication 和 authorization 等主要类别。
- 可选 Capella Agentic SAST 为独立多阶段子工作流。

## 3. 黑盒输入兼容性

决定性限制来自 CLI：

- `apps/cli/src/index.ts` 明确要求 `--url and --repo are required`；
- `apps/cli/src/help.ts` 将 `--repo` 标记为 required；
- `start.ts` 无条件解析和挂载 repo；
- Worker workflow 依赖 repoPath；
- recon prompt 明确要求把 live app 行为与完整源码关联。

所以没有源码仓库时，用户不能只传一个 URL 运行完整产品。

## 4. 条件性优势

如果同时有源码和测试站点，Shannon 具备：

- Playwright 浏览器；
- 登录流程、TOTP、邮箱 OTP 和 SSO 支持；
- 实际命令和浏览器利用；
- “no exploit, no report”硬性结果门；
- SAST 候选和动态候选对账去重；
- PDF、Markdown、JSON 和 SARIF 报告；
- Temporal 重试、恢复和失败隔离。

这些能力解释了它在灰盒场景中的高价值，但不能用于给黑盒 SRC 加分到第一梯队。

## 5. 黑盒短板

- 无 subfinder/httpx/amass/ASN 等大范围资产发现链。
- 偏单应用/单 URL，不适合多子域和多目标计划。
- 没有 Caido/Burp 式 MITM 流量证据系统。
- 业务逻辑覆盖受固定漏洞 lane 和源码驱动方式限制。
- `rules_of_engagement` 主要由提示词执行，缺每次出站前的强制 scope gate。
- 单次运行耗时和模型成本较高。

## 6. Scope 与安全性

Shannon 有受限源码沙箱和 Docker 隔离，但针对网络目标的 allowlist、重定向、速率、并发和危险动作审批没有形成像 T3MP3ST 那样的统一执行层硬门。真实 SRC 中仍需外围网络策略。

## 7. README 与实现差异

README 说它“分析源码、识别攻击路径并执行真实 exploit”，这一点与实现一致。问题不是宣传夸大，而是容易被“autonomous AI pentester”标签误认为支持纯黑盒；实际 CLI 明确要求源码。

## 8. 评分

| 维度 | 分数 |
|---|---:|
| 资产/子域/API 发现 | 20 |
| 浏览器/代理/认证态 | 70 |
| 业务逻辑 | 50 |
| 自动利用与 PoC | 75 |
| 误报控制 | 75 |
| 证据与报告 | 80 |
| Scope/限速/审批 | 55 |
| 多目标与扩展 | 25 |
| 部署成本 | 40 |

这些维度分反映“若满足输入条件”的实现质量。由于黑盒场景缺少必需 repo，最终综合分额外受到场景兼容性惩罚。

**最终判断：35/100。** 常规无源码 SRC 应直接排除；有源码和 staging 时另按灰盒工具评估。

## 9. 关键源码依据

- `apps/cli/src/index.ts`
- `apps/cli/src/help.ts`
- `apps/cli/src/commands/start.ts`
- `apps/worker/src/temporal/workflows.ts`
- `apps/worker/prompts/recon.txt`
- `apps/worker/src/services/validate-authentication.ts`
- `apps/worker/src/services/exploitation-checker.ts`
- `COVERAGE.md`
