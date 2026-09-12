# Run 13 立项书：托管模式正式满分局（行营）

> 2026-09-12，主调度草拟，用户已确认开工（"正式托管跑吧"）。
> 基础：run 12 试跑全链路验证通过（`VALIDATION/L4-RUN17553-17583-hosted-trial.md`）；
> 探索事实：`VALIDATION/HOSTED-MODE-EXPLORATION.md`；行营 POC 已固化（run 11 满分）。

## 1. 目标（字典序）

- **唯一硬指标：满分 40/40（20800）**。
- 次要 1：用时（拿满最后一题立即收尾停表）；次要 2：花费（官方 DS 通道，
  与本地同价，jisi 账本按平台口径）。
- 非目标：不追求并发调优实验；不做镜像新功能（v7 相对试跑 v6 仅：
  插件版本一致性同步 + 行营 preset + 正式开战令）。

## 2. 约束

- 硬：托管规则 6（clean-room，打包审计 0 违规）；凭据只走 env_config；
  镜像 ≤3GB；6h 平台钟；8核16G 沙箱（进程死=局终，无重启——稳健第一）；
  从零开始（严禁历史题解/flag/机制先验）。
- 软：行营模式（persona 三件套）；收割专员制等 run 11 调度经验由 SKILL/
  战报传承（不写死模型偏好，决策权在主 agent）。

## 3. 验收口径

1. 满分 40/40 → 立即收尾（xiaochang_finish + 进程退出）；
2. 平台 status=finished、final_flags=40；
3. 赛后 sessions/model-usage/run_events 可拉，hosted-replay 重建齐整
   （time 线/file-tree/deliverables/usage——含产物上移对话层检验）；
4. 行营三件套在托管局可见（preset 生效：立项书/计划评审/复盘落对话层）；
5. 花费落账（jisi_usage + 平台 model-usage 双口径对账）。

## 4. 规则域（红线前置）

- 红线 = §2。托管日志部分公开（对话原文审计）——prompt/文件内容视为
  可公开，不写敏感信息；publish 与否战后由用户裁定。

## 5. 假设日志

- 假设 1：v7 镜像 = v6 + 三处增量，试跑验证过的链路不回退（env 权限模式/
  F28 runner/SSRF 陷阱等保持）。
- 假设 2：行营 preset 在沙箱内挂载正常（join 补丁已在 -dev checkout 内）。
- 假设 3：8C16G 下 3 容器 + 4-6 并发执行者的调度可跑满 6h 不崩（主 agent
  据 jisi_usage/资源观察自行收敛）。
- 假设 4：结算走官方 DS 通道（平台网关观测模型名 deepseek-flash）。
- 假设 5：无托管 run 次数配额限制（试跑 6 局未触顶）。

## 6. 计划（DAG）

```
A 插件版本一致性同步(jisi/hufu/runner/compat/xingying lib) ─┐
B 行营 preset 进镜像 + agent-presets roster + DSH_PRESET      ├─ v7 镜像
C 正式开战令(run11 合规版托管适配)                           ┘
D 打包审计 → build → save(≤3GB) → COS 上传                  ← 依赖 A/B/C
E 建托管局(agent=神策 2779, env_config 三 key)              ← 依赖 D
F 全程值守(平台 status/WS/sessions; 异常就地处置)           ← 依赖 E
G 赛后 hosted-replay + L4 记录 + 复盘回填 + 榜前决策         ← 依赖 F
```

- 关键路径 A→D→E→F→G 串行；A/B/C 并行（同仓不同文件）。

## 7. 复盘回填区（2026-09-13 回填）

- **结果**：验证轮 run 17706 四项全过（fanout 9 模型/GLM 网关/原生 DS max 档/1 题入账）；
  正式局 run 17710 **36/40、18100 分（87.02%）、5h19m**——满分未达成。
- **修复项全部实战生效**（fanout 9×9、glm 执行者 1319 调用、autosubmit +2500）。
- **根因**：①停表线错位（内部 330min 预算 < 平台 360min，agent 提前 41 分钟收官，
  措辞是我写的）②g-24 被 fanout 的数学证明带进错误空间后误判"无解"（run 11 用
  单密钥 fake_admin 路线解出过）③单局方差（g-27 v7 解出、本局失败）。
- **教训分类**：A=停表条款必须以平台钟为唯一依据且内部预算归零也继续；B=无；
  C=本局战报/复盘已上移对话层可回放；D=无平台缺陷。
- **详见**：`VALIDATION/L4-RUN17710-live.md`。
