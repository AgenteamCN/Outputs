# HiveMQ Cloud vs EMQX Cloud（Chief <-> Human 专用高可靠通道）可行性调研报告

## 1. 关键发现：HiveMQ Cloud Serverless 已停止新建并将于 2026-12-31 退场
- 根据 HiveMQ 官方文档，**HiveMQ Cloud Serverless（免费版与按量付费版）将于 2026 年 12 月 31 日正式停止服务（EOL）**，且目前已禁止创建新的 Serverless 集群。
- 替代方案 **HiveMQ Cloud Starter** 为独立集群，**没有永久免费额度**（仅 $100 试用金，约 12 天 24x7 运行，随后 **$0.34/小时 ≈ $245/月** + $0.80/百万条消息），且**无中国大陆节点**。
- 结论：**不可采用 HiveMQ Cloud Serverless**，也不建议为 2~10 条 `chief <-> human` 连接支付 ~$245/月切换 HiveMQ Starter。

## 2. 对比矩阵
| 维度 | EMQX Cloud Serverless（当前） | HiveMQ Cloud Serverless | HiveMQ Cloud Starter（替代） |
|---|---|---|---|
| 状态 | 活跃 | 2026-12-31 EOL（不可新建） | 活跃（独立集群） |
| 免费额度 | 每月 100 万 session-min + 1 GB 流量 | 已停止新建 | 无永久免费（~$245/月起） |
| 协议与端口 | MQTT 3.1/5.0, TLS `8883`, WSS `8084` | MQTT 3.1.1/5.0, TLS `8883`, WSS `8884` | MQTT 3.1.1/5.0, TLS `8883`, WSS `8884` |
| 闲置断连 | 共享负载均衡约 ~285s 切断 | 多租户 Serverless | 独立 Broker 无共享 LB 285s 切断 |
| 大陆延迟 | 支持中国大陆低延迟节点 | 无大陆节点 | 无大陆节点 |

## 3. 推荐方案
1. **方案 A（当前零成本，推荐）**：继续使用 EMQX Cloud Serverless，结合已合入的 **#169（15s keepalive + `lan_server.rs` 规范 Close 帧）** 与 `agyWorker` DO SQLite 环形缓冲游标重放，已可平稳应对 ~285s 共享负载均衡断连。
2. **方案 B（低成本独立 Broker，~$5–15/月）**：若未来确实需要完全消除 285s 共享负载均衡切断，建议在轻量 VPS / Cloud Run 自建 **EMQX / Mosquitto / HiveMQ Enterprise Lab（25 连接免费许可）**，而非每月 $245 的 HiveMQ Cloud Starter。
