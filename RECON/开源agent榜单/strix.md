# strix 分析报告

- **能力范围**：全流程
- **SRC 适用度**：中——能对授权目标跑黑白盒 web/API 并产出 PoC；但偏单目标/代码扫描，缺大规模资产测绘与 SRC 合规流程，复杂登录态/反爬/业务逻辑需人工介入。

## 核心思路
用 LLM 多 Agent（根+动态派生专家子 Agent）在 Docker 沙箱内调用真实渗透工具（浏览器、Caido 代理、nuclei/sqlmap、源码分析）对授权目标做动态侦察、利用并以 PoC 验证漏洞，最终产出带修复建议的报告。

## 架构
单进程异步多 Agent 树：AgentCoordinator 维护父子关系/状态/信箱；openai-agents SDK Runner+SQLite 会话，agents.json 快照断点续扫；Docker Kali 沙箱工具链，Caido 代理中间人，budget/turn 双控。

## 技术面
黑盒 DAST(真实浏览器+Caido+subfinder/katana/nuclei/sqlmap/ffuf)；白盒源码感知(semgrep/ast-grep+依赖 CVE)；多 Agent 编排+信箱+wait 机制；RAG 技能库(~70 个漏洞/框架/协议 md)；MCP 接入+PoC 验证去重+CVSS/SARIF 报告。

## 短板
严重依赖模型质量与预算；覆盖与完成情况为 Agent 自报，可信度有限；无跨目标记忆；需显式授权+Docker；无凭证时业务逻辑/反爬/登录态覆盖弱。

## 值得神策借鉴
地址化 Agent 树+信箱/生命周期工具的可恢复多 Agent 协作协议；威胁模型/覆盖台账/去重等共享状态支撑'负空间'可信报告；统一 Docker 沙箱+Caido 代理让 Agent 用真实工具链。
