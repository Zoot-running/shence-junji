# OWASP Nettacker 分析报告

- **能力范围**：纯技术/工具
- **SRC 适用度**：中——适合 SRC 前置侦察/资产测绘、子域枚举、目录爆破、默认口令与已知 CVE 复核；但无法自主发现未知漏洞，缺利用与 PoC 验证。

## 核心思路
并非用 AI 挖洞，而是靠声明式 YAML 模板+正则/条件签名匹配+多协议多线程，自动化执行侦察、已知 CVE 核验与凭据爆破，全程确定性规则、无智能推理。

## 架构
CLI+Flask REST API/Web UI；multiprocess.Process 按目标分组+Thread 按(目标×模块)×请求并发；模块为 YAML(payloads→steps→response 条件正则)；SQLite/MySQL/PG 存事件；report 与 compare_report 输出并做历史漂移对比；port_scan 服务发现结果反向驱动后续模块。

## 技术面
YAML 声明式模块模板；正则/条件签名匹配；多协议库(http/ftp/ssh/smb/smtp/telnet/icmp/ssl/socket)；已知 CVE PoC 规则库；字典爆破与 nettacker_fuzzer；多进程多线程并发+socks 代理/随机 UA。

## 短板
无 AI 自主推理，规则全靠人工写 YAML；仅检测已知 CVE/签名，未知漏洞与业务逻辑盲区；正则易误漏报；无自主利用与修复建议闭环；大范围扫描负载重易被 WAF/IDS 拦截。

## 值得神策借鉴
声明式 YAML 模块模板低门槛沉淀攻击面规则；服务发现结果反向驱动后续模块(端口→协议→模块)的编排联动；DB 历史+漂移/对比报告支持持续监控与跨轮复用。
