# pentagi 分析报告

- **能力范围**：全流程
- **SRC 适用度**：中——通用黑盒渗透平台，可对授权目标跑真实工具并出报告，但缺 SRC 的资产/范围管控、限速防误伤、PoC 校验与报告模板，自主执行易误报与越权。

## 核心思路
以 primary_agent 为主控，将用户目标经 generator/refiner 拆解为 task→subtask，委派 pentester/coder/searcher 等专家子 agent，在 Kali Docker 沙箱内调用真实渗透工具自主完成侦察与利用，并借助向量记忆与上下文摘要生成漏洞报告。

## 架构
Go 后端(REST+GraphQL)+React 前端。主控拆 task→subtask 委派专家子 agent；工具在 Kali Docker 沙箱隔离执行；PostgreSQL+pgvector 长期记忆，csum 链式摘要控上下文，可选 Graphiti/Neo4j 知识图谱，Langfuse/OTel 观测。

## 技术面
分层多 agent 委派；Docker 沙箱真实工具(Kali: nmap/metasploit/sqlmap)；向量记忆 RAG+知识图谱；Web 情报检索(多搜索 API+只读 scraper)；链式摘要上下文+planner/mentor/reflector 监督防死循环。

## 短板
自主执行缺严格 scope/限速护栏；浏览器只读无交互点击；无 MCP 扩展；监督模式 2-3 倍 token/时延；无 SRC 化资产测绘/PoC 复核/报告格式。

## 值得神策借鉴
generator→refiner→reporter 任务分解与报告闭环；planner/mentor/reflector 多层监督+相同工具调用阈值防死循环；Docker 沙箱+pgvector 长期记忆+链式摘要这套基础设施。
