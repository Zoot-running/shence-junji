# L4 run 6（15998）现场记录 —— 架构校验重点（进行中）

> 开跑：2026-09-08 01:50（elapsed 从首个 OpenAPI 调用起算）。主 agent = dev headless（deepseek-v4-pro/max，全新会话）。
> 战前净化：归档 run 5 会话 + 快照 + usage sidecar；集思账本（deepseek-v4-flash 9/9）保留跨 run 继承。
> 价格表已校准（L2-MODEL-CATALOG-2026-09）。余额快照：DeepSeek ¥284.60 / Kimi ¥181.07 / 智谱 ¥159.92。

## 得分轨迹
| 时刻 | elapsed | 分数 | 事件 |
|---|---|---|---|
| 01:54 | ~4min | 0 | 两次启动失败修环境（F1-F4） |
| 02:07 | 17min | 2100 | g-39(hard+1000)/g-19(+500)/g-30(+600) |
| 02:13 | 24min | 4300 | 11 题 |
| 02:19 | 29min | 7500 | 18 题，容器满载 |
| 02:30 | 40min | 10500 | 24 题 |
| 02:33 | 43min | 11400 | 27 题，硬题区开火 |
| 02:41 | 52min | 14200 | 30 题，g-38/g-01/g-40 三 hard 已倒 |
| 02:45 | 56min | 16500 | 33 题 |
| 02:47 | 57min | 17200 | 34 题（g-24 落） |
| 02:52 | 62min | 17800 | 35 题（g-17 CSIDH MITM 干净解出） |
| 03:05 | 76min | 18600 | 37 题（风暴期间 g-34 执行者自交卷 +800） |
| 03:10 | 81min | 18600 | 收官 4 题（g-02/g-03/g-06/g-14）执行者在磨 |

花费轨迹：02:19 ¥6.83 → 02:30 ¥9.4 → 02:41 ¥13.7 → 02:45 ¥14.5 → 03:00 DeepSeek ¥264.31（¥20.3）/ Kimi ¥179.14（fanout 花 ¥1.93）/ 智谱少量（fanout+glm 执行者）。

## 发现定稿（F1-F17）

### A. 启动/运维（已全部修复）
- **F1** DSH_HOME 未导出 → headless 落错 home、无工具。修复：run6-launch.sh 显式 export。
- **F2** dev home 缺 API 凭据（MISSING_CREDENTIAL）→ boot exit 1 连挂。修复：launcher 导出三 key。应把凭据注入写进 jintuo 文档。
- **F3** self-pkill 再现（pkill -f 匹配自身 cmdline）。纪律：一律按 PID 杀。
- **F4** GUARD_CMD 需 export（普通赋值不随 exec 传播）。

### B. 治理/clean-room（本轮最大发现域）
- **F5** workdir 携带 run 5 全部题解（260 文件：g-* 目录/solve 脚本/fe_*.js）。clean-room 门禁只扫 knowledgeDir，不扫 workdir。已清理归档。改进：开跑前 sweep workdir 或每 run 独立 workdir。
- **F6** 虎符战报跨 run 残留（cwd/boards/<组>/FINDINGS.md 42 份旧战报带入 run 6 首波）。改进：boards 按 run 隔离或开跑前清空。
- **F7** 归档位置可见性：先放 .dsh-dev 树内，tool-less boot 的 agent 自行翻查。已移出 + 开战令加红线。
- **F12** 平台 flag 跨 run 不变（g-39/g-13/g-32 与 run 5 相同）→ 历史记忆复用风险极高，clean-room 必须严格。
- **F14** 污染窗口实锤：波 1-12 主 agent 明确引用"上 run 完整解法+现成脚本"；02:17 清理后（wave 13+）零引用。→ **前 27 分钟成绩含旧工件红利，收官阶段才是纯架构验证**。

### C. 工具/运行时
- **F8** glm 派单静默失败（3/3，detail 为空）：子代理 header 实锤误路由 `provider=deepseek-official, model=glm-5.3-flash, effort=low` → DeepSeek 模型名校验错误 → 静默死亡。对照实验（独立进程 fanout 带/不带 effort）均成功；02:50 fanout 与 03:10 glm 执行者在战役进程内均成功 → **误路由是开局 4 分钟窗口的暂态**（疑插件注册竞态），叠加"解析失败静默回落默认路由"bug 造成 3 连死亡。修复已落地：jisi `0bfa3d9`（解析不到 provider/目录未宣告 → 响亮失败）+ runner `ca2be63`（enqueue 目录校验）。
- **F9** usage sidecar 盲区：只覆盖 compat 路由；native deepseek 不记（backlog）；战役前期 glm 未路由成功 → usageLines 长期 0。复现进程/战役 fanout 均正常落盘 → 机制本身没问题。
- **F10** watch-campaign 缺 AUDIT_SEEN 门（启动期误报 stale 999999s）。已修复（jintuo `9520900`）。
- **F11** agent 的 bash 看不到 BENCHMARK_TOKEN（平台知识应经工具提供）。
- **F13** continuable 0 使用；fanout 收官 hard 双子首次使用（5 模型并行征思路，主 agent 原话 cheap insurance）——fanout 形态合理；continuable 待复盘（board 传递 vs 续战省轮次）。

### D. 守护链（live 风暴，全部已修复）
- **F15** runner 审计心跳是活动驱动的（无独立定时器）→ fanout 卡 kimi-k3 慢模型 10 分钟 → 心跳停摆。修复：120s 真心跳定时器（yebushou `8fb4493`）。
- **F16** guard AUDIT_SEEN 判定 bug（"文件存在"=现任写过；旧 mtime 仍算）→ stale-kill 重启后新进程无 boot 宽限 → 每 20-30s 连杀的重启风暴。修复：mtime ≥ CHILD_STARTED 判定（jintuo `9316044`）。
- **F17** guard 只杀 bash 包装层 → node 孤儿（12562）存活并发跑。修复：setsid 进程组杀（jintuo `76fdd11`）。
- 风暴期间架构设计经受住检验：执行者为独立子进程，主 agent 进程被 kill 后 g-34 执行者照常交卷（+800）；goal-rearm 使新进程无损续跑。

### E. 正向验证
- 集思账本生效（12 个首批派单 9 个选 deepseek-v4-flash 9/9）；glm 3 连失败后主 agent 主动弃用（负反馈闭环）。
- CTF 公理生效（g-39 附件先取）。
- 派单可靠性：38+ verdict 中 failed 仅 glm×3（pre-fix），round-timeout 0 触发。
- 花费纪律：贵模型（kimi-k3/k2.6 大调用）仅 fanout 时用一次；主力 deepseek 夜间半价 + glm-flash。
- hint 0 使用（主 agent 一直自研/战报磨题）。

## 战后待办
1. 确认 xiaochang_finish 停表（主 agent 应自动执行；否则手动调前端 API）。
2. dev headless profile 重装插件（jisi `0bfa3d9` + runner `ca2be63`/`8fb4493` 进入已装构建）：
   ```bash
   cd /mnt/d/Software/WSLSoftware/Agents/deepseek-harness-dev
   node apps/cli/lib/bin.js plugin --profile headless rm @shence/jisi
   node apps/cli/lib/bin.js plugin --profile headless add "file:/mnt/d/Software/WSLSoftware/Projects/Security/ShenCe/shence-jisi"
   node apps/cli/lib/bin.js plugin --profile headless rm @shence/xiaochang-runner
   node apps/cli/lib/bin.js plugin --profile headless add "file:/mnt/d/Software/WSLSoftware/Projects/Security/ShenCe/shence-yebushou/packages/runner"
   ```
3. F13 复盘：continuable 零使用的判断（board 传递 vs 续战）。
4. 终稿 L4-RUN15998.md：得分曲线 vs run 5、花费对比、F1-F17 定稿。
5. 余额校准快照复核。
6. 启动规程固化：run6-launch.sh 的经验（DSH_HOME/凭据/export/按 PID 杀）应进 jintuo README。
