# L4 run 6（15998）现场记录 —— 架构校验重点（进行中）

> 开跑：2026-09-08 01:50（elapsed 从首个 OpenAPI 调用起算）。主 agent = dev headless（deepseek-v4-pro/max，全新会话）。
> 战前净化：归档 run 5 会话 + 快照 + usage sidecar；集思账本（deepseek-v4-flash 9/9）保留跨 run 继承。
> 价格表已校准（L2-MODEL-CATALOG-2026-09）。余额快照：DeepSeek ¥284.60 / Kimi ¥181.07 / 智谱 ¥159.92。

## 得分轨迹（live）
- 01:54（~4min）：0 分 —— 两次启动失败在修环境（见 F1-F4）。
- 02:07（17min）：2100 / 3 题（g-39 hard +1000、g-19 +500、g-30 +600）。
- 02:13（24min）：4300 / 11 题。
- 02:19（29min）：7500 / 18 题，容器 2-3 满载，DeepSeek 花费 ¥6.83，Kimi/智谱零花费。
- 02:30（40min）：10500 / 24 题，容器 3 满载，DeepSeek ¥9.4，Kimi/智谱仍零花费。
- 02:33（43min）：11400 / 27 题。
- 02:41（52min）：14200 / 30 题。
- 02:45（55min）：15700 / 33 题，硬题逐一下落（+500/波）。
- 02:45（56min）：16500 / 33 题。平台实况剩 7 题（g-02/g-03/g-06/g-14/g-17/g-24/g-34，共 4300 分），容器已清空、主 agent 排下一波。g-38/g-01/g-40 三 hard 已倒（各磨 11-12 轮）。DeepSeek ¥13.7。
- **F8/F9 旁证**：独立复现进程的 jisi_fanout(glm) 成功调用在 sidecar 落 2 行（usageLines 0→2）——sidecar 机制本身正常，只覆盖 compat 路由；战役进程全程 0 行 = glm 派单从未成功路由过（与 F8 同源）。硬题区开火：g-38/g-01/g-25 三硬核在打（wave 11，各 ~11 rounds 磨题），主 agent 已排好 wave 12-15 全量计划（g-40/g-27/g-28 → g-02/g-03/g-34 → g-24/g-17/g-06 → g-14）。
- **观察 F13（候选）**：硬题磨题仍用一次性 deepseek-v4-flash 重派（rounds:11 重开新兵），continuable 续战机制 0 使用；战报（FINDINGS.md）承担了跨轮状态传递。战后判断：continuable 未激活是主 agent 判断（board 传递够用）还是机制未达发现门槛。
- 派单模型分布：deepseek-v4-flash ×25（全成功）、glm-5.3-flash ×3（全静默失败，见 F8）——主 agent 已停止使用 glm（账本/观察生效）。

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
- **F14 污染窗口实锤（F5 延伸）**：主 agent 波 1-12 的执行者 prompt 明确引用"上 run 完整解法 + 现成脚本 /home/zrn/xiaochang-work/g-XX/solve_live.py"——即 02:17 清理前，主 agent 主动使用了 run 5 工件。清理后（wave 13+）最新会话无任何遗留引用 → **前 27 分钟成绩含旧工件红利；02:17 之后的收官阶段才是纯架构验证**。另：hufu_continue/jisi_fanout 工具描述确认在主 agent 工具表里（机制可达），F13 是"可达但未用"，属调度偏好问题。
- **F12 平台 flag 跨 run 不变**：g-39/g-13/g-32 的 flag 与 run 5 完全一致（平台不重随机化）。→ 任何历史 flag 记忆 = 直接复用；clean-room 价值比预想更高，F5/F6 的清理是必须而非可选。

### 工具/运行时类
- **F8 glm-5.3-flash 派单静默失败（根因已实锤）**：子代理 session 的 request/header 显示 `provider=deepseek-official, model=glm-5.3-flash, reasoningEffort=low` → DeepSeek 报"supported API model names are..." → 子代理无输出静默死亡（turn/end reason=error 但 detail 落账为空）。即：**model→provider 解析在战役进程里失效，回落到父 agent 的 deepseek-official 路由**。对照实验：独立 workdir 新进程 jisi_fanout(glm-5.3-flash) 带/不带 effort=low 均成功——同一 profile 新 boot 正常，战役进程异常（进程内状态不可事后内省）。修复（已实施，run 内不生效、下次 boot 生效）：①jisi.delegate 解析不到 provider / 目录未宣告模型 → 响亮失败（jisi `0bfa3d9`，已测 26/26）；②runner enqueue 前校验模型在集思目录（yebushou `ca2be63`，已测 10/10）；③hufu terminal detail 带诊断——jisi 已有 [diagnostic] 追加逻辑，空 detail 的根因是 DSH 对 model-name 校验错误不产 diagnostic，①的护栏使该场景不再发生。
- **F9 usage sidecar 零写入（根因已定位，与 F8 同源）**：全部会话检索无任何 zhipu 调用痕迹 → glm 子代理在 spawn 阶段即失败，从未产生 LLM 调用；usage sidecar 只覆盖 compat 路由（kimi/智谱），native deepseek 路由本就不记（backlog：deepseek-native-adapter 也要 sidecar）。→ 战役全程花费核算盲区，靠 watcher 余额差兜底。
- **F10 watch-campaign 误报**：审计文件尚不存在时报 stale 999999s（启动期 2 条误警）。guard-runner 有 AUDIT_SEEN 门，watcher 没有。**已修复**（shence-jintuo `9520900`，AUDIT_SEEN 门，watcher 已重启）。
- **F11 agent 的 bash 里看不到 BENCHMARK_TOKEN**：main agent 曾为探平台 API 花 2 轮逆向前端（token 在 launcher env，不在 agent shell）。平台知识应经工具而非 agent shell 提供。

### 正向观察
- 集思账本生效：12 个 enqueue 里 9 个选 deepseek-v4-flash（账本 9/9）——按历史胜率+价格选型符合设计。
- CTF 公理生效：g-39 先取附件解出（"decoded from this run's attachment"）。
- 花费纪律：7500 分只花 DeepSeek ¥6.83（夜间半价窗口），贵模型零调用。
- 全新会话（无 run 5 记忆）依然 29 分钟 18 题——但受 F5/F6 污染影响，前半程含旧工件红利，后程才是纯架构验证。

## 战后待办（收尾清单）
1. 确认 xiaochang_finish 停表（主 agent 应自动执行；否则我手动调前端 API）。
2. dev headless profile 重装插件（jisi `0bfa3d9` + runner `ca2be63` 的 F8 护栏进入已装构建）。
3. F13 复盘：continuable/fanout 全程零使用——查主 agent 是否知道这两个机制（工具描述可达性），判断是机制门槛还是合理判断。
4. 终稿：得分曲线 vs run 5 对比、花费对比、F1-F12 定稿 → L4-RUN15998.md。
5. 余额校准快照（DeepSeek/Kimi/智谱）与 jisi priceTable 一致性复核。
