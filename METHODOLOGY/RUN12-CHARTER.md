# Run 12 立项书：托管模式试跑（链路验证局，非满分局）

> 2026-09-12，主调度草拟，用户已确认开工（"小规模试跑吧"）。
> 前置三件套已验收（HOSTED-PREP-CHARTER）；托管探索事实见 VALIDATION/HOSTED-MODE-EXPLORATION.md。

## 1. 目标（字典序）

- **唯一硬指标：托管链路端到端跑通**——镜像上传→沙箱拉起我们的 agent→VPN/网关可用→
  解出并提交 **≥1 题**（产生得分事件）→赛后 sessions/model-usage/run_events 可拉、
  hosted-replay 重建器跑通。
- 非目标：满分、用时、排名（正式托管局另立项）；不追求并发调优。

## 2. 约束

- 硬：镜像 ≤3GB；clean-room（规则 6，打包审计 0 违规）；凭据全部 env_config 注入、
  镜像零密钥；只用 set 5（open 题集，closed 的 set 4 不碰）；平台协议 7.2（不逆向）。
- 软：**靠配置不靠纪律**——.gw 网关地址写死在镜像配置里，agent 不用改任何东西；
  试跑预算：起跑后 ~30 分钟内完成验证并 finish（省平台时钟，留正式局额度）。

## 3. 验收口径（done 定义）

1. `hosted-pack-audit.py` 对镜像内容目录 0 违规；
2. 镜像 tar.gz 上传成功（upload-cred + COS 直传），≤3GB；
3. `POST /runs {run_source:'hosted', app_dist, env_config}` 建局成功，沙箱拉起
   headless DSH（进程日志可见：VPN 预检通过、网关 LLM 调用成功）；
4. 平台得分事件 ≥1 题（answer_correct + score_delta>0）；
5. `xiaochang_finish` 后平台 status=finished；
6. 赛后 `llm/sessions`、`llm/model-usage`、`run_events` 可拉，`hosted-replay.py`
   重建产物齐整（timeline/file-tree/usage-summary）。

## 4. 规则域（红线前置）

- 红线 = §2 硬约束。试跑局允许低分（目标不是分），但**不允许**违规内容进镜像
  （打包审计为硬门）；值守期间发现平台异常 → 先留证再处置（题必有解纪律的
  运维版：链路失败先排查配置，不轻易判"平台不支持"）。

## 5. 假设日志

- 假设 1：沙箱内平台 API（BENCHMARK_BASE_URL）与 VPN（10.0.100.58）由平台预置可达。
- 假设 2：网关 `http://api.deepseek.com.tsecbench.gw/v1` 走 OpenAI 兼容协议、
  BYO key 即可用；Kimi/GLM 网关同构（试跑主用 flash，Kimi/GLM 仅 fanout 备用）。
- 假设 3：DSH 原生 deepseek-official 适配器 baseURL 不可配置 → 全部路由改用
  llm-openai-compat（provider 重命名 dsgw），settings.yaml 指向 dsgw。
- 假设 4：托管 run 无次数配额限制（quota_exhausted 仅活动赛题）；若有配额冲突，
  试跑局即终止并留证。
- 假设 5：试跑镜像不带行营 preset（最小变量面）；正式局再带（persona 已在
  本地验证过，属纯增量）。

## 6. 计划（DAG + 关键路径）

```
A1 装 docker(WSL apt docker.io + 起 dockerd)      ─┐
A2 镜像构建上下文(精简 checkout + profile 骨架      ├─ A 环境与镜像
   + .gw 路由配置 + entrypoint + 合规开战令)        ┘
B 打包审计 → docker build → save|gzip(≤3GB)        ← 依赖 A
C 上传(upload-cred → COS 直传) → 建托管 run        ← 依赖 B
D 值守(平台 status/WS/告警; 异常就地处置)          ← 依赖 C
E 赛后 hosted-replay 重建 + L4 记录 + 复盘回填      ← 依赖 D
```

- 关键路径 A→B→C→D→E 串行；A1 与 A2 并行（docker 装、上下文组装互不依赖）。
- 并行度：A1∥A2；其余串行链。

## 7. 复盘回填区（战后）
