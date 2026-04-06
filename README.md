# Phantun WireGuard Docker

## 项目简介

基于 **WireGuard + Phantun + Docker** 构建的企业级跨国异地组网方案，解决 UDP 被运营商拦截、跨国丢包、频繁断连等问题。

## 解决的问题

| 问题 | 解决方案 |
|:-----|:---------|
| UDP 被运营商拦截 | Phantun 将 UDP 封装为 TCP 443 |
| 连接成功率 ≤50% | 实现 **100%** 连通率 |
| 延迟高 (200ms+) | 优化至 **45-60ms** |
| 频繁断连 | 容器自动重启，7×24小时稳定运行 |

## 技术架构

**核心原理：双容器共享网络命名空间 + TCP 封装 WireGuard 流量**

```
美国客户端：WireGuard → Phantun 客户端（UDP→TCP）→ 公网（TCP 443）
国内服务端：Phantun 服务端（TCP→UDP）→ WireGuard → 公司内网
```

## 核心组件

| 组件 | 作用 | 部署位置 |
|:-----|:-----|:---------|
| wg-access-server | WireGuard 服务端 + Web 管理界面 | Docker 容器 A（国内） |
| Phantun 服务端 | TCP 流量解封装回 UDP | Docker 容器 B（国内） |
| Phantun 客户端 | WireGuard UDP 流量封装为 TCP | 美国客户端 |
| FortiGate 防火墙 | 仅放行 TCP 443 和 8888 | 公司网络边界 |

## 端口规划

| 端口 | 协议 | 用途 | 暴露范围 |
|:-----|:-----|:-----|:---------|
| 443 | TCP | 公网隧道入口（伪装 HTTPS） | 全球可访问 |
| 8888 | TCP | Web 管理界面 | 公司内网 |
| 51820 | UDP | 容器内部 WireGuard 通信 | 仅本地回环 |

## 流量路径

```
美国办公电脑
    ↓ (WireGuard 产生 UDP 51820 流量)
Phantun 客户端（UDP → TCP 封装）
    ↓ (TCP 443 包)
═══════════════════════════════════
           互联网（跨国）
═══════════════════════════════════
    ↓
FortiGate 防火墙（放行 TCP 443）
    ↓
Docker 宿主服务器
    ↓
Phantun 服务端（TCP → UDP 解封装）
    ↓ (共享网络命名空间，127.0.0.1)
wg-access-server（解密 WireGuard 流量）
    ↓
公司内网资源
```

## 方案迭代历程

| 版本 | 技术方案 | 协议 | 成功率 | 问题 |
|:-----|:---------|:-----|:-------|:-----|
| 一 | WireGuard + Mimic（MT3000） | UDP | ≤50% | UDP 穿透差 |
| 二 | WireGuard + UDPTunnel | UDP | ≤70% | 仍受 UDP 限制 |
| 三 | WireGuard + Phantun（MT3000） | TCP | 100% | 路由器性能差 |
| 四 | 单容器 wg-access-server | UDP | ≤60% | UDP 被封杀 |
| 五 | 单容器（优化端口） | UDP | ≤75% | 无法突破 UDP 问题 |
| **六** | **双容器（WG + Phantun）** | **TCP** | **100%** | ✅ 最终方案 |

**关键转折：从"优化 UDP"转向"TCP 封装"**

## 快速部署

### 服务端（上海总部）

```bash
# 启动 WireGuard 容器
docker run -d \
  --name wg-access-server \
  --restart always \
  --cap-add NET_ADMIN \
  --cap-add SYS_MODULE \
  -p 8888:8000/tcp \
  -v /etc/wireguard:/etc/wireguard \
  -e WG_ADMIN_PASSWORD=your_password \
  -e WG_LISTEN_PORT=51820 \
  -e WG_VPN_CIDR=10.8.0.0/24 \
  weejewel/wg-access-server

# 启动 Phantun 容器（共享网络命名空间）
docker run -d \
  --name phantun-server \
  --restart always \
  --cap-add NET_ADMIN \
  --network container:wg-access-server \
  ghcr.io/dndx/phantun:latest \
  phantun-server -l 0.0.0.0:443 -r 127.0.0.1:51820 -k "your_key" --mtu 1400
```

### 客户端（北美）

```bash
# 运行 Phantun 客户端
phantun-client -l 127.0.0.1:51820 -r YOUR_SERVER_IP:443 -k "your_key"

# WireGuard 配置
[Interface]
PrivateKey = <your-private-key>
Address = 10.8.0.2/24
MTU = 1400

[Peer]
PublicKey = <server-public-key>
Endpoint = 127.0.0.1:51820
AllowedIPs = 192.168.0.0/24, 10.8.0.0/24
PersistentKeepalive = 20
```

## 性能指标

| 指标 | 数值 |
|:-----|:-----|
| 连通率 | 100% |
| 延迟 | 45-60ms |
| 丢包率 | ≤0.05% |
| 运行时间 | 7×24小时连续 |

## 项目成果

### 业务成果
- 北美测试人员可 7×24 小时稳定访问内网，效率提升 30%
- 光模块测试可视化 Web 工具跨网使用率 100%
- 相比商业 VPN 方案，年节省授权费用约 8 万元

### 技术成果
- 形成可复用的企业级跨国组网模板
- 输出《部署手册》《排查手册》《扩展指南》3 份文档
- 验证了"TCP 封装解决 UDP 限制""双容器共享网络栈"等关键技术

### 安全合规
- 双重加密：Phantun 传输加密 + WireGuard 隧道加密
- 防火墙严格限制源地址与端口
- 通过内部安全合规审核

## 经验总结

### 技术选型
- 先用轻量载体（MT3000）快速验证假设，再投入容器化

### 跨国组网避坑
- 避免依赖 UDP，优先 TCP 封装或成熟 VPN 协议
- 对外暴露常用端口（443、8888）降低拦截概率
- 容器化部署优先于硬件部署

## 后续扩展

- 多出口冗余：增加备用公网线路 + DNS 轮询
- 集群高可用：多节点 + Keepalived VIP
- 监控告警：Prometheus + Grafana、ELK 日志分析
- 权限管理：RBAC 角色权限、操作审计

## 适用场景

- 企业总部 ↔ 海外分公司互联
- 跨国研发、测试、办公组网
- UDP 被限制、传统 VPN 不稳定环境
- 中小规模企业生产环境

## 项目角色

架构设计、容器部署、端口规划、网络调试与外网连通性验证

---

**上海孛璞半导体技术有限公司**
