# L4 run 6（15998）现场记录 —— 架构校验重点（进行中）

> 开跑：2026-09-08 01:50（elapsed 从首个 OpenAPI 调用起算）。主 agent = dev headless（deepseek-v4-pro/max，全新会话）。
> 战前净化：归档 run 5 会话 + 快照 + usage sidecar；集思账本（deepseek-v4-flash 9/9）保留跨 run 继承。
> 价格表已校准（L2-MODEL-CATALOG-2026-09）。余额快照：DeepSeek ¥284.60 / Kimi ¥181.07 / 智谱 ¥159.92。

## 得分轨迹（live）
- 01:54（~4min）：0 分 —— 两次启动失败在修环境（见 F1-F4）。
- 02:07（17min）：2100 / 3 题（g-39 hard +1000、g-19 +500、g-30 +600）。
- 02:13（24min）：4300 / 11 题。
- 02:19（29min）：7500 / 18 题，容器 2-3 满载，DeepSeek 花费 ¥6.83，Kimi/智谱零花费。

## 发现（F 编号，战后定稿）

### 启动/运维类
- **F1 DSH_HOME 未导出**：guard-runner 拉起 headless 时只传了 GUARD_DSH_HOME，子进程 DSH_HOME 落错 home → 无工具可用，agent 空转 4 分钟找工具（还去翻了归档会话）。→ run6-launch.sh 显式 `export DSH_HOME` 修复。
- **F2 dev home 缺凭据**：MISSING_CREDENTIAL deepseek-official 致 boot 直接 exit 1（guard 连挂 2 次）。dev 的 .credentials.yaml 只有浏览器会话 grant；生产用 refs。→ launcher 导出 DEEPSEEK/KIMI/ZHIPU_API_KEY 修复。**应把凭据注入写进 jintuo/guard 的启动文档与脚本**。
- **F3 self-pkill 再现**：`pkill -f "guard-runner.sh"` 匹配到自身 shell cmdline 被杀，新 guard 没起来。→ 一律按 PID 杀。
- **F4 GUARD_CMD 需 export**：脚本里普通赋值不随 `exec bash guard-runner.sh` 传播，guard 报 GUARD_CMD required。→ export 修复。

### 治理/clean-room 类（本轮最大发现域）
- **F5 工作目录携带上一 run 全部题解**：/home/zrn/xiaochang-work 里有 run 5 的 260 个遗留文件（g-* 每题工作目录、solve 脚本、fe_*.js 前端 dump）。clean-room 门禁只扫 knowledgeDir，**不扫 workdir**。战前未清。已把 mtime<01:47 的遗留全部移入 /home/zrn/xiaochang-archive/workdir-run5/（保留 .venv/.gocache/.gopath 工具链）。**改进：开新 run 前 sweep workdir，或每 run 用独立 workdir。**
- **F6 虎符战报跨 run 残留**：board 路径 = cwd/boards/<组>/FINDINGS.md，run 5 的 42 份 FINDINGS 原样带进 run 6，首波执行者直接读到旧战报（main agent 波计划里明说 "board solutions"）。已清（旧 mtime 的 FINDINGS 覆盖为占位）。**改进：boards 按 run 隔离或开跑前清空。**
- **F7 归档位置可见性**：先前的会话归档放在 .dsh-dev 树内，tool-less boot 的 agent 自行探索了归档。已移到 /home/zrn/xiaochang-archive 并在开战令加"禁止读归档"红线。**改进：归档落 DSH_HOME 之外 + 开战令默认含红线。**
- **F12 平台 flag 跨 run 不变**：g-39/g-13/g-32 的 flag 与 run 5 完全一致（平台不重随机化）。→ 任何历史 flag 记忆 = 直接复用；clean-room 价值比预想更高，F5/F6 的清理是必须而非可选。

### 工具/运行时类
- **F8 glm-5.3-flash 派单静默失败**：3/3 次 dispatch（g-19/g-30/g-32）terminal=failed 且 detail 为空；主 agent 自己兜底解掉（verdict complete）。直接 curl 验证 API 正常（thinking enabled / 不带 thinking 字段均可）。根因待查（疑 zhipu-official 派单链路或 effort 处理）；**detail 为空 = 失败诊断未透出，audit 需要补 diagnostic**。
- **F9 usage sidecar 零写入**：usageLines:0，dev storages 无 llm-usage.jsonl；已装 compat 构建含写入代码（adapter.ts）。jisi_usage 全程无数据 → 花费核算盲区。待查：headless spawn 的子代理 LLM 调用是否走 compat stream，或写盘路径漂移。
- **F10 watch-campaign 误报**：审计文件尚不存在时报 stale 999999s（启动期 2 条误警）。guard-runner 有 AUDIT_SEEN 门，watcher 没有。→ 补同样门。
- **F11 agent 的 bash 里看不到 BENCHMARK_TOKEN**：main agent 曾为探平台 API 花 2 轮逆向前端（token 在 launcher env，不在 agent shell）。平台知识应经工具而非 agent shell 提供。

### 正向观察
- 集思账本生效：12 个 enqueue 里 9 个选 deepseek-v4-flash（账本 9/9）——按历史胜率+价格选型符合设计。
- CTF 公理生效：g-39 先取附件解出（"decoded from this run's attachment"）。
- 花费纪律：7500 分只花 DeepSeek ¥6.83（夜间半价窗口），贵模型零调用。
- 全新会话（无 run 5 记忆）依然 29 分钟 18 题——但受 F5/F6 污染影响，前半程含旧工件红利，后程才是纯架构验证。
