# 🌐 IPv6 过渡技术完整教程

> Comprehensive IPv6 Transition Technologies Tutorial — NAT64 / DNS64 / Dual Stack / Tunneling

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![RFC 6146](https://img.shields.io/badge/RFC-6146-red)](https://tools.ietf.org/html/rfc6146)
[![RFC 6147](https://img.shields.io/badge/RFC-6147-red)](https://tools.ietf.org/html/rfc6147)

---

## 📖 目录 / Table of Contents

- [一、背景与原理 / Background & Principles](#一背景与原理--background--principles)
  - [IPv4 地址枯竭 / IPv4 Address Exhaustion](#ipv4-地址枯竭--ipv4-address-exhaustion)
  - [IPv6 特性 / IPv6 Features](#ipv6-特性--ipv6-features)
  - [过渡技术概览 / Transition Technologies Overview](#过渡技术概览--transition-technologies-overview)
- [二、双栈部署 / Dual-Stack Deployment](#二双栈部署--dual-stack-deployment)
- [三、隧道技术 / Tunneling Technologies](#三隧道技术--tunneling-technologies)
- [四、NAT64 + DNS64 部署 / NAT64 + DNS64 Deployment](#四nat64--dns64-部署--nat64--dns64-deployment)
  - [原理 / Principles](#原理--principles)
  - [Docker 部署 / Docker Deployment](#docker-部署--docker-deployment-1)
  - [原生安装 / Native Installation](#原生安装--native-installation-1)
  - [NAT64 网关配置示例 / NAT64 Gateway Configuration](#nat64-网关配置示例--nat64-gateway-configuration)
- [五、46XLAT (CLAT) / 46XLAT (CLAT)](#五46xlat-clat--46xlat-clat)
- [六、SIIT / Stateless IP/ICMP Translation](#六siit--stateless-ipicmp-translation)
- [七、常用命令 / Common Commands](#七常用命令--common-commands)
- [八、常见问题 / FAQ](#八常见问题--faq)
- [九、生产环境最佳实践 / Production Best Practices](#九生产环境最佳实践--production-best-practices)
- [☕ 支持 / Support](#-支持--support)

---

## 一、背景与原理 / Background & Principles

### IPv4 地址枯竭 / IPv4 Address Exhaustion

全球 IPv4 地址池已于 2019 年 11 月由 RIPE NCC 分配完毕。这意味着**新的网络接入无法获得公网 IPv4 地址**。IPv6 是唯一的长期解决方案。

The global IPv4 address pool was fully allocated by RIPE NCC in November 2019. This means **new network connections cannot obtain public IPv4 addresses**. IPv6 is the only long-term solution.

| 地区 / Region | IPv4 耗尽时间 / Exhaustion | IPv6 部署率 / Deployment Rate |
|---|---|---|
| APNIC (亚太) | 2011 年 4 月 | ~50% |
| RIPE (欧洲) | 2019 年 11 月 | ~35% |
| ARIN (北美) | 2015 年 9 月 | ~45% |
| LACNIC (拉美) | 2020 年 6 月 | ~15% |
| AFRINIC (非洲) | 预计 2025 年 | ~5% |

### IPv6 特性 / IPv6 Features

| 特性 / Feature | IPv4 | IPv6 |
|---|---|---|
| 地址长度 / Address Length | 32 位 (4 字节) | 128 位 (16 字节) |
| 地址总数 / Total Addresses | ~43 亿 | ~3.4×10³⁸ |
| 地址配置 / Address Config | DHCP / 手动 | SLAAC / DHCPv6 / 手动 |
| NAT | 普遍需要 / Commonly needed | 不需要 / Not needed |
| 安全特性 / Security | IPsec 可选 | IPsec 内建要求 |
| 广播 / Broadcast | 有 | 无 (用多播替代) |
| 分片 / Fragmentation | 路由器和发送端 | 仅发送端 |
| 校验和 / Checksum | 有 | 无 (依赖上层) |
| 选项字段 / Options | 可变长度头 | 扩展头链 |

### 过渡技术概览 / Transition Technologies Overview

```
┌─────────────────────────────────────────────────────────┐
│                   IPv6 过渡技术图谱                       │
│              IPv6 Transition Technology Map              │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌── 双栈 / Dual Stack ───────────── 最直接的方法       │
│  │  (RFC 4213 - Basic Transition Mechanisms)            │
│  │                                                      │
│  ├── 隧道 / Tunneling ────────────── 利用现有 IPv4 网络 │
│  │  ├─ 6to4 (RFC 3056)                                 │
│  │  ├─ Teredo (RFC 4380)                               │
│  │  ├─ ISATAP (RFC 5214)                               │
│  │  └─ GRE/6in4/SIT                                    │
│  │                                                      │
│  ├── 翻译 / Translation ────────── IPv6 ↔ IPv4 协议转换 │
│  │  ├─ NAT64 + DNS64 (RFC 6146, 6147) ★ 推荐           │
│  │  ├─ 464XLAT (RFC 6877) ★ 移动端推荐                 │
│  │  └─ SIIT (RFC 7915)                                 │
│  │                                                      │
│  └── 卸载 / Offloading ─────────── 运营商级解决方案     │
│     ├─ MAP-E (RFC 7597)                                │
│     ├─ MAP-T (RFC 7599)                                │
│     └─ Lightweight 4over6 (RFC 7596)                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 二、双栈部署 / Dual-Stack Deployment

双栈（Dual-Stack）是最简单、最成熟的过渡技术。网络设备同时运行 IPv4 和 IPv6 协议栈，DNS 根据可用性返回合适的地址。

Dual-Stack is the simplest and most mature transition technology. Network devices run both IPv4 and IPv6 protocol stacks simultaneously, with DNS returning appropriate addresses based on availability.

### 检查系统双栈支持 / Check System Dual-Stack Support

```bash
# 检查 IPv6 是否启用 / Check if IPv6 is enabled
cat /proc/sys/net/ipv6/conf/all/disable_ipv6
# 输出 0 表示启用 / Output 0 means enabled

# 查看接口 IPv6 地址 / View interface IPv6 addresses
ip -6 addr show

# 检查双栈连通性 / Check dual-stack connectivity
ping -4 -c 3 google.com
ping -6 -c 3 google.com
```

### Linux 启用双栈 / Enable Dual-Stack on Linux

```bash
# 查看当前状态 / Check current status
sysctl net.ipv6.conf.all.disable_ipv6

# 启用 IPv6 (如果禁用) / Enable IPv6 (if disabled)
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=0
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=0

# 持久化配置 / Make persistent
echo "net.ipv6.conf.all.disable_ipv6=0" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6=0" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# 配置 /etc/network/interfaces (Debian/Ubuntu)
sudo cat >> /etc/network/interfaces << 'EOF'

# IPv6 配置 / IPv6 Configuration
iface eth0 inet6 static
    address 2001:db8:1234::100
    netmask 64
    gateway 2001:db8:1234::1
EOF

# 或使用 netplan (Ubuntu 18+)
sudo cat > /etc/netplan/01-netcfg.yaml << 'EOF'
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp6: true
      accept-ra: true
EOF
sudo netplan apply
```

### Docker 双栈配置 / Docker Dual-Stack Configuration

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "2001:db8:1::/80",
  "experimental": true,
  "ip6tables": true
}
```

```bash
# 写入 Docker 配置 / Write Docker config
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "ipv6": true,
  "fixed-cidr-v6": "2001:db8:1::/80",
  "experimental": true,
  "ip6tables": true
}
EOF

# 重启 Docker / Restart Docker
sudo systemctl restart docker

# 验证 IPv6 支持 / Verify IPv6 support
docker run --rm alpine ping6 -c 2 google.com

# 创建双栈网络 / Create dual-stack network
docker network create --ipv6 --subnet "172.20.0.0/16" --subnet "2001:db8:2::/64" my-net
```

---

## 三、隧道技术 / Tunneling Technologies

### 6in4（SIT 隧道） / 6in4 (SIT Tunnel)

将 IPv6 包封装在 IPv4 包中传输，需要公网 IPv4 地址。

Encapsulates IPv6 packets inside IPv4 packets, requires a public IPv4 address.

```bash
# 创建 SIT 隧道 / Create SIT tunnel
sudo ip tunnel add sit1 mode sit remote 203.0.113.1 local 198.51.100.1
sudo ip link set sit1 up

# 配置 IPv6 地址 / Configure IPv6 address
sudo ip -6 addr add 2001:db8:aaaa::1/64 dev sit1

# 添加 IPv6 路由 / Add IPv6 route
sudo ip -6 route add ::/0 dev sit1

# 使用 HE TunnelBroker (免费) / Using HE TunnelBroker (free)
# 注册: https://tunnelbroker.net
sudo ip tunnel add he-ipv6 mode sit remote 216.218.221.6 local YOUR_IPV4 ttl 255
sudo ip link set he-ipv6 up
sudo ip -6 addr add YOUR_ASSIGNED_IPV6/64 dev he-ipv6
sudo ip -6 route add ::/0 dev he-ipv6

# 持久化 / Make persistent (Ubuntu Netplan)
sudo cat > /etc/netplan/10-tunnel.yaml << 'EOF'
network:
  version: 2
  tunnels:
    he-ipv6:
      mode: sit
      remote: 216.218.221.6
      local: YOUR_IPV4
      ttl: 255
      addresses:
        - "YOUR_ASSIGNED_IPV6/64"
      routes:
        - to: "::/0"
EOF
sudo netplan apply
```

### GRE 隧道 / GRE Tunnel

通用路由封装（GRE）可以封装多种协议（包括 IPv6）。

Generic Routing Encapsulation (GRE) can encapsulate multiple protocols (including IPv6).

```bash
# 创建 GRE 隧道 / Create GRE tunnel
sudo ip tunnel add gre1 mode gre remote 203.0.113.1 local 198.51.100.1 ttl 255
sudo ip link set gre1 up
sudo ip -6 addr add 2001:db8:bbbb::1/64 dev gre1
sudo ip -6 route add 2001:db8:cccc::/64 via 2001:db8:bbbb::2 dev gre1

# 删除隧道 / Delete tunnel
sudo ip tunnel del gre1
```

### Teredo / Teredo Tunneling

Teredo 是一种在 NAT 后的 IPv4 网络中提供 IPv6 连接的隧道协议。

Teredo is a tunneling protocol that provides IPv6 connectivity behind NAT IPv4 networks.

```bash
# Ubuntu/Debian 安装 Miredo / Install Miredo
sudo apt install -y miredo

# 启动后自动获得 2001:0000:/32 地址
# Automatically obtains 2001:0000:/32 address after start
sudo systemctl enable --now miredo

# 检查 Teredo 地址 / Check Teredo address
ip -6 addr show teredo
ping6 ipv6.google.com

# 配置文件 / Configuration file
cat /etc/miredo/miredo.conf
# 默认使用 teredo.remlab.net 服务器
# Default uses teredo.remlab.net server
```

---

## 四、NAT64 + DNS64 部署 / NAT64 + DNS64 Deployment

### 原理 / Principles

**NAT64** 是一种 IPv6 → IPv4 的协议转换技术。当一个 IPv6 主机要访问 IPv4 网络时，NAT64 网关将 IPv6 包翻译为 IPv4 包（反之亦然）。它需要与 **DNS64** 配合使用。

NAT64 is an IPv6-to-IPv4 protocol translation technology. When an IPv6 host needs to access an IPv4 network, the NAT64 gateway translates IPv6 packets to IPv4 packets (and vice versa). It works together with **DNS64**.

**工作流程 / Workflow**:

```
IPv6 Client → DNS64 server → 合成 AAAA 记录 (合成 IPv6 地址) 
→ 连接合成地址 → NAT64 网关 → 转换为 IPv4 连接 → IPv4 Server
```

**DNS64 合成规则 / DNS64 Synthesis Rule**:

当查询一个仅有 A 记录（IPv4）的域名时，DNS64 会合成一个 AAAA 记录，其前缀为预定义的 IPv6 前缀（默认 64:ff9b::/96）。

When querying a domain with only A records (IPv4), DNS64 synthesizes an AAAA record with a predefined IPv6 prefix (default 64:ff9b::/96).

```
合成地址示例 / Synthesized Address Example:
原始 IPv4: 203.0.113.5
合成 IPv6: 64:ff9b::cb00:7105  (即 64:ff9b::203.0.113.5)
```

### Docker 部署 / Docker Deployment

#### 使用 dns64/nat64 容器方案 / Using dns64/nat64 Container Setup

```bash
# 1. 创建 Docker 网络 / Create Docker network
docker network create --ipv6 --subnet "172.20.0.0/16" --subnet "2001:db8:100::/64" nat64-net

# 2. 部署 DNS64 (基于 Unbound) / Deploy DNS64 (based on Unbound)
docker run -d --name dns64 \
  --network nat64-net \
  --restart unless-stopped \
  -p 53:53/udp \
  -e DNS64_PREFIX=64:ff9b::/96 \
  registry.cn-hangzhou.aliyuncs.com/n42/dns64:latest
# 或使用官方 Unbound 自行配置
# Or use official Unbound with custom config

# 3. 部署 NAT64 (基于 TAYGA) / Deploy NAT64 (based on TAYGA)
docker run -d --name nat64 \
  --network nat64-net \
  --restart unless-stopped \
  --cap-add=NET_ADMIN \
  -e NAT64_PREFIX=64:ff9b::/96 \
  -e IPV4_ADDR=172.20.0.10 \
  -e IPV6_ADDR=2001:db8:100::10 \
  towhands/nat64:latest
```

#### 使用 Jool (内核级 NAT64) 容器 / Using Jool (Kernel-level NAT64) Container

```bash
# Jool 是 Linux 内核模块，推荐用原生安装
# Jool is a Linux kernel module, native install recommended
# 这里提供 Docker 方式 / Docker approach provided here
docker pull nicmx/jool

# 创建 NAT64 配置 / Create NAT64 config
cat > /tmp/jool.conf << 'EOF'
{
  "pool6": "64:ff9b::/96",
  "pool4": "192.168.1.100-192.168.1.200",
  "disabled": false
}
EOF

# 运行 Jool / Run Jool
docker run --rm --privileged --net=host \
  -v /tmp/jool.conf:/etc/jool/jool.conf \
  nicmx/jool jool instance add "nat64" /etc/jool/jool.conf
```

#### 完整 Docker Compose 方案 / Full Docker Compose Setup

```yaml
version: '3.8'

services:
  # DNS64 服务器 / DNS64 Server
  dns64:
    image: jafarlihi/unbound-dns64:latest
    container_name: dns64
    restart: unless-stopped
    ports:
      - "53:53/udp"
      - "53:53/tcp"
    environment:
      - DNS64_PREFIX=64:ff9b::/96
    cap_add:
      - NET_ADMIN

  # NAT64 网关 / NAT64 Gateway (TAYGA)
  nat64:
    image: towhands/nat64:latest
    container_name: nat64
    restart: unless-stopped
    network_mode: host
    cap_add:
      - NET_ADMIN
      - NET_RAW
    environment:
      - NAT64_PREFIX=64:ff9b::/96
      - IPV4_ADDR=0.0.0.0
      - IPV6_ADDR=2001:db8:100::1

  # 测试客户端 / Test Client
  test-client:
    image: alpine:latest
    container_name: ipv6-test
    command: sh -c "apk add --no-cache bind-tools && tail -f /dev/null"
    networks:
      - nat64-net
    dns:
      - 172.20.0.2

networks:
  nat64-net:
    driver: bridge
    enable_ipv6: true
    ipam:
      config:
        - subnet: 172.20.0.0/16
        - subnet: 2001:db8:100::/64
```

```bash
# 启动所有服务 / Start all services
docker-compose up -d

# 测试 IPv6 → IPv4 连接 / Test IPv6 → IPv4 connectivity
docker exec ipv6-test ping6 -c 3 64:ff9b::8.8.8.8

# 测试 DNS64 / Test DNS64
docker exec ipv6-test nslookup google.com 172.20.0.2
```

### 原生安装 / Native Installation

#### TAYGA (NAT64 用户态实现)

```bash
# Ubuntu/Debian 安装 / Install on Ubuntu/Debian
sudo apt update
sudo apt install -y tayga

# 配置 TAYGA / Configure TAYGA
sudo tee /etc/tayga.conf > /dev/null << 'EOF'
tun-device nat64
ipv4-addr 192.168.255.1
prefix 64:ff9b::/96
dynamic-pool 192.168.255.0/24
data-dir /var/lib/tayga
EOF

# 启动 TAYGA / Start TAYGA
sudo systemctl enable --now tayga

# 添加路由 / Add routes
sudo ip route add 192.168.255.0/24 dev nat64
sudo ip -6 route add 64:ff9b::/96 dev nat64

# 测试 / Test
ping6 64:ff9b::8.8.8.8
```

#### Jool (NAT64 内核模块) — 推荐 / Jool (Kernel Module) — Recommended

```bash
# Ubuntu/Debian 安装 / Install on Ubuntu/Debian
sudo apt update
sudo apt install -y jool-dkms jool-tools

# 加载内核模块 / Load kernel module
sudo modprobe jool_siit
sudo modprobe jool_nat64

# 创建 NAT64 实例 / Create NAT64 instance
sudo jool instance add "nat64" --pool6 64:ff9b::/96

# 添加 IPv4 池 (可用的转换地址) / Add IPv4 pool (translatable addresses)
sudo jool -i "nat64" pool4 add 192.168.1.100 192.168.1.200

# 查看状态 / Check status
sudo jool -i "nat64" instance display
sudo jool -i "nat64" pool6 display
sudo jool -i "nat64" pool4 display

# 查看会话 / View sessions
sudo jool -i "nat64" session display

# 启用自动启动 / Enable on boot
echo "jool_siit" | sudo tee -a /etc/modules
echo "jool_nat64" | sudo tee -a /etc/modules
```

#### Unbound DNS64

```bash
# 安装 Unbound / Install Unbound
sudo apt install -y unbound

# 配置 DNS64 / Configure DNS64
sudo tee /etc/unbound/unbound.conf.d/dns64.conf > /dev/null << 'EOF'
server:
    interface: 0.0.0.0
    port: 53
    access-control: 0.0.0.0/0 allow
    access-control: ::0/0 allow
    
    # DNS64 配置 / DNS64 Configuration
    module-config: "dns64 validator iterator"
    dns64-prefix: 64:ff9b::/96
    
    # 上游 DNS / Upstream DNS
    forward-zone:
        name: "."
        forward-addr: 8.8.8.8
        forward-addr: 1.1.1.1
EOF

# 启动 Unbound / Start Unbound
sudo systemctl enable --now unbound

# 验证 DNS64 / Verify DNS64
dig @localhost AAAA google.com
# 应返回一个合成 AAAA 记录 (64:ff9b::...)
# Should return a synthesized AAAA record
```

### NAT64 网关配置示例 / NAT64 Gateway Configuration

#### 单臂 NAT64 / One-arm NAT64

NAT64 网关只有一个网络接口，流量通过路由策略引导。

The NAT64 gateway has a single network interface; traffic is routed via policy-based routing.

```bash
# 创建 TAYGA 虚拟接口 / Create TAYGA virtual interface
sudo tayga --mktun
sudo ip addr add 192.168.255.1 dev nat64
sudo ip addr add 64:ff9b::/96 dev nat64
sudo ip link set nat64 up

# 添加 IPv6 → NAT64 路由 / Add IPv6 → NAT64 route
sudo ip -6 route add 64:ff9b::/96 dev nat64

# 启用 IPv4 转发 / Enable IPv4 forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# 添加 NAT iptables 规则 / Add NAT iptables rules
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i nat64 -j ACCEPT
sudo iptables -A FORWARD -o nat64 -j ACCEPT
```

#### 双臂 NAT64 / Two-arm NAT64

NAT64 网关有两个接口，分别连接 IPv6 和 IPv4 网络。

The NAT64 gateway has two interfaces, one for IPv6 network and one for IPv4 network.

```bash
# 接口分配 / Interface assignment
# eth0 — IPv4 网络 / IPv4 network (192.168.1.0/24)
# eth1 — IPv6 网络 / IPv6 network (2001:db8:1::/64)

# 配置 TAYGA / Configure TAYGA
cat > /etc/tayga.conf << 'EOF'
tun-device nat64
ipv4-addr 192.168.255.1
ipv6-addr 2001:db8:ffff::1
prefix 64:ff9b::/96
dynamic-pool 192.168.255.0/24
data-dir /var/lib/tayga
EOF

# 启动并配置路由 / Start and configure routing
sudo systemctl restart tayga
sudo ip route add 192.168.255.0/24 dev nat64
sudo ip -6 route add 64:ff9b::/96 dev nat64
sudo ip -6 route add 2001:db8:ffff::/64 dev nat64

# 转发规则 / Forwarding rules
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1

# IPv4 NAT 规则 / IPv4 NAT rules
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i nat64 -o eth0 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o nat64 -j ACCEPT
```

---

## 五、46XLAT (CLAT) / 46XLAT (CLAT)

464XLAT (RFC 6877) 是移动运营商广泛使用的过渡技术，它结合了 **CLAT** (客户侧翻译器) 和 **PLAT** (运营商侧翻译器，即 NAT64)。

464XLAT (RFC 6877) is a transition technology widely used by mobile carriers, combining **CLAT** (Customer-side Translator) and **PLAT** (Provider-side Translator, i.e., NAT64).

### 架构 / Architecture

```
App → CLAT (客户侧翻译/Customer-side) → IPv6 网络 → PLAT/NAT64 → IPv4 互联网
```

```bash
# Android 设备上可通过此项查看 CLAT 状态
# Check CLAT status on Android
adb shell dumpsys connectivity | grep -A 10 CLAT
adb shell ip -6 addr show clat4

# Linux 上使用 clatd 工具
# Use clatd on Linux
sudo apt install -y clatd

# 配置 clatd / Configure clatd
sudo tee /etc/clatd.conf > /dev/null << 'EOF'
# CLAT IPv6 地址前缀 / CLAT IPv6 address prefix
plat_prefix 64:ff9b::/96

# CLAT IPv4 地址 / CLAT IPv4 address
clat_ipv4 192.0.0.4

# 上游 DNS / Upstream DNS
dns64_server 2001:db8:ff::1

# 启用 DNS64 合成 / Enable DNS64 synthesis
dns64_enabled yes
EOF

# 启动 clatd / Start clatd
sudo systemctl enable --now clatd

# 验证 / Verify
ip addr show clat
ping 8.8.8.8 -I clat
```

---

## 六、SIIT / Stateless IP/ICMP Translation

SIIT (Stateless IP/ICMP Translation, RFC 7915) 是一种**无状态**的 IP 头翻译技术，不需要维护会话状态。

SIIT is a **stateless** IP header translation technology that doesn't maintain session state.

```bash
# 使用 Jool SIIT / Using Jool SIIT
sudo modprobe jool_siit

# 创建 SIIT 实例 / Create SIIT instance
sudo jool_siit instance add "siit" --pool6 64:ff9b::/96

# 添加 EAMT (显式地址映射表) / Add EAMT (Explicit Address Mapping Table)
# IPv4 1.2.3.4 → IPv6 2001:db8:1234::1
sudo jool_siit eamt add 1.2.3.4 2001:db8:1234::1

# 批量映射 / Batch mapping
sudo jool_siit eamt add 10.0.0.0/24 2001:db8:aaaa::
sudo jool_siit pool6791v6 add 64:ff9b::/96

# 查看映射 / View mappings
sudo jool_siit eamt display

# 删除 SIIT 实例 / Delete SIIT instance
sudo jool_siit instance remove "siit"
```

---

## 七、常用命令 / Common Commands

### IPv6 诊断命令速查表 / IPv6 Diagnostic Command Cheat Sheet

| 命令 / Command | 说明 / Description |
|---|---|
| `ip -6 addr show` | 查看 IPv6 地址 / Show IPv6 addresses |
| `ip -6 route show` | 查看 IPv6 路由 / Show IPv6 routes |
| `ip -6 neigh show` | 查看 IPv6 邻居表 (NDP) / Show IPv6 neighbor table |
| `ping6` 或 `ping -6` | IPv6 Ping 测试 |
| `traceroute6` 或 `mtr -6` | IPv6 路由追踪 / IPv6 traceroute |
| `ss -6 -tan` | 查看所有 IPv6 TCP 连接 / Show all IPv6 TCP connections |
| `ip6tables -L -n` | 查看 IPv6 防火墙规则 / View IPv6 firewall rules |
| `dig AAAA example.com` | 查询 AAAA 记录 / Query AAAA records |
| `nslookup -type=AAAA example.com` | 同上 / Same as above |
| `curl -6 http://example.com` | 强制使用 IPv6 访问 / Force IPv6 access |
| `wget -6 http://example.com` | 同上 / Same as above |
| `sysctl net.ipv6.conf.all.disable_ipv6` | 检查 IPv6 是否禁用 / Check if IPv6 is disabled |

### IPv6 地址常用格式 / Common IPv6 Address Formats

| 格式 / Format | 示例 / Example |
|---|---|
| 完整 / Full | `2001:0db8:85a3:0000:0000:8a2e:0370:7334` |
| 压缩 / Compressed | `2001:db8:85a3::8a2e:370:7334` |
| IPv4 映射 / IPv4-mapped | `::ffff:192.168.1.1` |
| NAT64 前缀 / NAT64 Prefix | `64:ff9b::192.168.1.1` |
| 链路本地 / Link-local | `fe80::1` |
| 多播 / Multicast | `ff02::1` (所有节点 / All nodes) |
| 环回 / Loopback | `::1` |
| 未指定 / Unspecified | `::` |

### NDP 邻居发现 / NDP Neighbor Discovery

```bash
# 查看 IPv6 邻居表 / View IPv6 neighbor table
ip -6 neigh show

# 手动添加邻居 / Manually add neighbor
ip -6 neigh add 2001:db8::2 lladdr 00:11:22:33:44:55 dev eth0 nud reachable

# 清除邻居表 / Flush neighbor table
ip -6 neigh flush all

# 修改 NDP 参数 / Modify NDP parameters
# 查看当前参数 / View current parameters
sysctl net.ipv6.neigh.eth0.base_reachable_time_ms
sysctl net.ipv6.neigh.eth0.gc_thresh1
sysctl net.ipv6.neigh.eth0.gc_thresh2
sysctl net.ipv6.neigh.eth0.gc_thresh3

# 调整邻居表大小 / Adjust neighbor table size
sudo sysctl -w net.ipv6.neigh.default.gc_thresh3=4096
sudo sysctl -w net.ipv6.neigh.default.gc_thresh2=2048
sudo sysctl -w net.ipv6.neigh.default.gc_thresh1=1024
```

### IPv6 防火墙 (ip6tables) / IPv6 Firewall (ip6tables)

```bash
# 基本规则 / Basic rules
sudo ip6tables -P INPUT DROP
sudo ip6tables -P FORWARD DROP
sudo ip6tables -P OUTPUT ACCEPT

# 允许已建立的连接 / Allow established connections
sudo ip6tables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 允许本地回环 / Allow loopback
sudo ip6tables -A INPUT -i lo -j ACCEPT

# 允许 ICMPv6 (必需!) / Allow ICMPv6 (required!)
sudo ip6tables -A INPUT -p ipv6-icmp -j ACCEPT

# 允许 NDP 邻居发现 / Allow NDP neighbor discovery
sudo ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 135 -j ACCEPT
sudo ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 136 -j ACCEPT

# 允许 SSH / Allow SSH
sudo ip6tables -A INPUT -p tcp --dport 22 -j ACCEPT

# 允许 DHCPv6 / Allow DHCPv6
sudo ip6tables -A INPUT -p udp --dport 546 -j ACCEPT
sudo ip6tables -A INPUT -p udp --dport 547 -j ACCEPT

# 保存规则 / Save rules
sudo ip6tables-save | sudo tee /etc/ip6tables.rules
```

---

## 八、常见问题 / FAQ

### Q1: 如何检查 ISP 是否提供 IPv6？

**方法 / Methods**:

```bash
# 方法 1: 检查 WAN 接口 / Check WAN interface
ip -6 addr show
# 如果有 2000::/3 范围内的地址，则有 IPv6
# If you have an address in 2000::/3 range, IPv6 is available

# 方法 2: 测试连通性 / Test connectivity
ping6 -c 2 2001:4860:4860::8888  # Google DNS IPv6

# 方法 3: 访问测试网站 / Visit test site
curl -6 https://test-ipv6.com

# 方法 4: 检查 DHCPv6 前缀 / Check DHCPv6 prefix
# 查看 /var/log/syslog 中的 dhclient 日志
grep dhclient /var/log/syslog | grep -i "IA_PD"
```

### Q2: NAT64 的地址前缀可以自定义吗？

**可以。** 推荐使用 `64:ff9b::/96` (IANA 分配)，但也可以使用自己的 /96 前缀。

**Yes.** The recommended prefix is `64:ff9b::/96` (IANA-assigned), but you can use your own /96 prefix.

```bash
# 使用自定义前缀 / Use custom prefix
# 例如 / Example: 2001:db8:9999::/96
sudo jool instance add "my-nat64" --pool6 2001:db8:9999::/96
sudo jool -i "my-nat64" pool4 add 10.0.0.100 10.0.0.200

# DNS64 也要配合修改 / DNS64 also needs to be updated
# 修改 Unbound 配置 / Update Unbound config
# dns64-prefix: 2001:db8:9999::/96
```

### Q3: IPv6 过渡技术对比 / Transition Technology Comparison

| 技术 / Tech | 优点 / Pros | 缺点 / Cons | 适用场景 / Use Case |
|---|---|---|---|
| 双栈 / Dual-Stack | 最简单，完全兼容 / Simplest, fully compatible | 需双份地址 / Need dual addresses | 所有场景 / All scenarios |
| 6in4 / 隧道 | 可穿越 IPv4 网络 / Can traverse IPv4 | 需公网 IPv4 / Need public IPv4 | 个人/小企业 |
| NAT64+DNS64 | 仅需 IPv6 客户端 / IPv6-only client | 单点故障 / Single point of failure | 运营商/校园网 |
| 464XLAT | 移动端最优 / Best for mobile | 部署复杂 / Complex deployment | 移动网络 / Mobile |
| SIIT | 无状态，高性能 / Stateless, high perf | 需地址映射表 / Needs address mapping | 数据中心 / DC |
| MAP-E/T | 运营商级 / Carrier-grade | 设备支持有限 / Limited device support | ISP 大网 / ISP |

### Q4: Docker 容器如何获取 IPv6 地址？

```bash
# 1. 确保 Docker 开启 IPv6 / Ensure Docker has IPv6 enabled
# 参考上文 Docker 双栈配置 / See Docker dual-stack config above

# 2. 创建 IPv6 网络 / Create IPv6 network
docker network create --ipv6 \
  --subnet "172.20.0.0/16" \
  --subnet "2001:db8:100::/64" \
  --gateway "2001:db8:100::1" \
  v6net

# 3. 在 IPv6 网络中启动容器 / Start container in IPv6 network
docker run -d --name web6 --network v6net nginx

# 4. 检查容器 IPv6 地址 / Check container IPv6 address
docker inspect web6 --format '{{range .NetworkSettings.Networks}}{{.GlobalIPv6Address}}{{end}}'

# 5. 分配固定 IPv6 地址 / Assign fixed IPv6 address
docker run -d --name web6-fixed \
  --network v6net \
  --ip6 "2001:db8:100::42" \
  nginx
```

### Q5: NAT64 如何处理 ICMPv6 → ICMPv4 转换？

NAT64 自动翻译 ICMPv6 错误消息为对应的 ICMPv4 错误消息，例如：

NAT64 automatically translates ICMPv6 error messages to corresponding ICMPv4 error messages:

| ICMPv6 类型 / Type | ICMPv4 类型 / Type | 说明 / Description |
|---|---|---|
| 1 (Destination Unreachable) | 3 (Destination Unreachable) | 目标不可达 |
| 2 (Packet Too Big) | 4 (Fragmentation Needed) | 包太大 / MTU 问题 |
| 3 (Time Exceeded) | 11 (Time Exceeded) | TTL 超时 |
| 4 (Parameter Problem) | 12 (Parameter Problem) | 参数错误 |

### Q6: 如何测试 IPv6 就绪程度 / How to test IPv6 readiness？

```bash
# 在线测试 / Online tests
# https://test-ipv6.com
# https://ipv6-test.com

# 命令行测试 / CLI tests
# 1. 同时测试 IPv4 和 IPv6
curl -s https://api.ipify.org && echo ""
curl -6 -s https://api6.ipify.org && echo ""

# 2. 测试 DNS64 功能 / Test DNS64 functionality
# 配置 IPv6-only DNS 后 / After configuring IPv6-only DNS:
dig @2001:db8:ff::1 AAAA ipv4only.example.com

# 3. 检查 IPv6 首选 / Check IPv6 preference
curl -s https://www.google.com -o /dev/null -w "%{remote_ip}"
# 如果返回 IPv6 地址，则 IPv6 已优先
# If returns IPv6 address, IPv6 is preferred

# 4. MTU 路径发现测试 / MTU path discovery test
ping6 -c 3 -M do -s 1452 ipv6.google.com
```

### Q7: 如何排查 NAT64 连接问题 / How to troubleshoot NAT64 connection issues？

```bash
# 步骤 1: 确认 DNS64 工作正常 / Verify DNS64 is working
dig @YOUR_DNS64 AAAA google.com
# 应返回 64:ff9b:: 开头的合成地址
# Should return synthesized address starting with 64:ff9b::

# 步骤 2: 测试直接 NAT64 连接 / Test direct NAT64 connection
ping6 -c 3 64:ff9b::8.8.8.8

# 步骤 3: 测试 TCP 连接 / Test TCP connection
curl -6 http://64:ff9b::1.1.1.1

# 步骤 4: 检查 NAT64 会话 / Check NAT64 sessions
sudo jool -i "nat64" session display

# 步骤 5: 检查日志 / Check logs
sudo journalctl -u jool -n 50
dmesg | grep -i "jool\|nat64\|tayga"
```

### Q8: 为什么我的应用在 IPv6-only 环境下无法工作？

**常见原因 / Common reasons**:

1. **硬编码 IPv4 地址** — 应用配置中写死了 IPv4 地址
   **解决**: 使用域名代替 IP / Use domain names instead of IPs

2. **IPv4 socket 绑定** — 应用只绑定了 AF_INET
   **解决**: 修改代码支持 AF_INET6 双栈 / Modify code to support AF_INET6 dual-stack

3. **NAT64 不支持的应用层协议** — FTP, SIP, RTSP 等
   **解决**: 使用 ALG (应用层网关) / Use ALG (Application Layer Gateway)

4. **DNSSEC 问题** — DNSSEC 签名与 DNS64 合成冲突
   **解决**: 配置 DNS64 绕过 DNSSEC 验证 / Configure DNS64 to bypass DNSSEC

---

## 九、生产环境最佳实践 / Production Best Practices

### 部署检查清单 / Deployment Checklist

- [ ] IPv6 地址规划已完成 / IPv6 address plan completed
- [ ] 双栈已对所有服务器启用 / Dual-stack enabled on all servers
- [ ] DNS 支持 AAAA 记录 / DNS supports AAAA records
- [ ] NAT64 网关双机热备 / NAT64 gateway HA pair
- [ ] 监控告警已配置 / Monitoring & alerting configured
- [ ] 安全策略已更新 (ip6tables) / Security policies updated
- [ ] 应用层兼容性已验证 / Application compatibility verified
- [ ] MTU 问题已排查 (PMTUD) / MTU issues addressed
- [ ] 回退机制已建立 (仅 IPv4 回退) / Fallback mechanism established

### 高可用 NAT64 双机部署 / HA NAT64 Deployment

```bash
# 使用 Keepalived 实现 VIP 漂移 / VIP failover with Keepalived
sudo apt install -y keepalived

# 主节点配置 / Primary node config
sudo tee /etc/keepalived/keepalived.conf > /dev/null << 'EOF'
global_defs {
    router_id NAT64_MASTER
}

vrrp_instance NAT64_VIP {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass nat64secret
    }
    virtual_ipaddress {
        192.168.1.254/24
        2001:db8:1::ff/64
    }
}
EOF

# 备份节点配置 / Backup node config
# state BACKUP, priority 50

# 启动 / Start
sudo systemctl enable --now keepalived
```

### 性能优化 / Performance Optimization

| 优化项 / Optimization | 配置 / Configuration | 效果 / Effect |
|---|---|---|
| 连接跟踪大小 / Conntrack Size | `net.netfilter.nf_conntrack_max=524288` | 支持更多 NAT64 会话 |
| IPv6 邻居表 / ND Table | `net.ipv6.neigh.default.gc_thresh3=8192` | 减少邻居表溢出 |
| Jool 线程数 / Jool Threads | `jool instance update nat64 --max-threads 4` | 提升并发 |
| TAYGA 缓冲 / TAYGA Buffer | 修改 tayga.conf: `buffer-size 65536` | 减少丢包 |
| 绑定 CPU / CPU Pinning | 使用 `taskset` 绑定 Jool/TAYGA 到专用核心 | 降低延迟 |

### 监控与日志 / Monitoring & Logging

```bash
# 监控 NAT64 活动会话数 / Monitor NAT64 active sessions
sudo jool -i "nat64" session display | wc -l

# 监控 Jool 统计信息 / Jool statistics
sudo jool -i "nat64" stats display

# Prometheus + Grafana 监控 / Prometheus + Grafana monitoring
# 编写 exporter 脚本 / Write exporter script
cat > /usr/local/bin/nat64_exporter.sh << 'SCRIPT'
#!/bin/bash
SESSION_COUNT=$(sudo jool -i "nat64" session display 2>/dev/null | wc -l)
echo "jool_active_sessions $SESSION_COUNT"
# 添加更多指标 / Add more metrics
SCRIPT
chmod +x /usr/local/bin/nat64_exporter.sh

# 设置 cron 采集 / Set up cron collection
echo "* * * * * /usr/local/bin/nat64_exporter.sh | curl -s -X POST --data-binary @- http://pushgateway:9091/metrics/job/nat64" | crontab -
```

---

## 📚 参考资料 / References

- [RFC 6146 — Stateful NAT64](https://tools.ietf.org/html/rfc6146)
- [RFC 6147 — DNS64](https://tools.ietf.org/html/rfc6147)
- [RFC 6877 — 464XLAT](https://tools.ietf.org/html/rfc6877)
- [RFC 7915 — SIIT](https://tools.ietf.org/html/rfc7915)
- [RFC 4213 — Basic Transition Mechanisms](https://tools.ietf.org/html/rfc4213)
- [Jool Documentation](https://nicmx.github.io/Jool/)
- [TAYGA — NAT64 for Linux](https://github.com/nccgroup/tayga)
- [IPv6 测试 / IPv6 Test](https://test-ipv6.com)
- [Hurricane Electric TunnelBroker](https://tunnelbroker.net)
- [Google IPv6 统计 / Google IPv6 Statistics](https://www.google.com/intl/en/ipv6/statistics.html)

---

## ☕ 支持 / Support

如果这个教程对你有帮助，欢迎请我喝杯咖啡：

**USDT (TRC20)**
```
TVbQerV1SF4MXB1JCcAzQxarewHwEPYTKm
```
