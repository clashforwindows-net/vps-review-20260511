# VPS 自建 DNS 解析体系与隐私过滤实战：Unbound / AdGuard Home / DoH·DoT

> DNS 是互联网的「总机」，也是运营商劫持、广告跟踪、DNS 污染的重灾区。本仓库教你用一台 VPS 自建完整 DNS 体系：Unbound 递归解析去中心化、AdGuard Home 全家去广告、DoH/DoT 加密防窃听、PowerDNS 托管权威域名，并给出分流策略、性能调优与一键部署脚本。

## 目录

- [为什么要自建 DNS](#为什么要自建-dns)
- [DNS 基础：递归、权威与转发](#dns-基础递归权威与转发)
- [架构总览：三种自建形态](#架构总览三种自建形态)
- [方案 A：Unbound 纯递归（隐私最强）](#方案-aunbound-纯递归隐私最强)
- [方案 B：AdGuard Home 全家桶（去广告+加速）](#方案-badguard-home-全家桶去广告加速)
- [方案 C：PowerDNS 权威托管（自管域名）](#方案-cpowerdns-权威托管自管域名)
- [DoH / DoT / DoQ 加密出口配置](#doh--dot--doq-加密出口配置)
- [分流策略：国内直连与国外加密](#分流策略国内直连与国外加密)
- [性能调优与缓存策略](#性能调优与缓存策略)
- [安全与防泄漏](#安全与防泄漏)
- [客户端接入：路由器与系统设置](#客户端接入路由器与系统设置)
- [一键部署与监控脚本](#一键部署与监控脚本)
- [常见问题 FAQ](#常见问题-faq)
- [相关资源与推荐入口](#相关资源与推荐入口)
- [免责声明](#免责声明)

## 为什么要自建 DNS

| 痛点 | ISP 默认 DNS | 自建 DNS 方案 |
|------|-------------|--------------|
| 域名污染/投毒 | 常见（尤其跨境场景） | 加密信道 + 权威源，可规避 |
| 隐私 | 运营商可记录全部查询 | 查询留在自己服务器 |
| 广告/跟踪 | 无法过滤 | AdGuard 规则实时拦截 |
| 劫持跳转 | 时有发生 | 校验 DNSSEC，防篡改 |
| 解析速度 | 取决于 ISP | 大缓存 + 就近递归，常更快 |

## DNS 基础：递归、权威与转发

先厘清三个角色，避免配置时概念混淆：

- **递归解析器（Recursive）**：替客户端从根域名一路问到结果，如 Unbound。
- **权威服务器（Authoritative）**：持有某域名最终记录，如你域名的 NS 指向 PowerDNS。
- **转发器（Forwarder）**：自己不递归，把请求转给上游，如 dnsmasq 默认行为。

自建体系通常组合使用：**AdGuard Home（面向客户端的入口+过滤+缓存）→ 上游走 Unbound（本地递归，摆脱第三方）**；需要自管域名解析时再叠加 PowerDNS 做权威。

## 架构总览：三种自建形态

```
形态一（推荐，单机两步）：
客户端 → AdGuard Home :53/:853（过滤+缓存）→ Unbound :5335（递归）→ 根服务器
                                                    ↑ 自带 DNSSEC 校验

形态二（极简，纯 Unbound）：
客户端 → Unbound :53（递归+DNSSEC）→ 根服务器

形态三（自管域名，三件套）：
公网 → PowerDNS :53（权威，托管 example.com）
内网 → AdGuard Home → Unbound（普通上网解析）
```

> 云 VPS 场景注意：若仅自用，**别把 53 端口直接暴露公网**，用防火墙限定来源 IP 或走 Tailscale/WireGuard 内网。

## 方案 A：Unbound 纯递归（隐私最强）

### 安装与最小配置

```bash
apt update && apt install -y unbound
```

编辑 `/etc/unbound/unbound.conf`：

```ini
server:
    # 只监听本机与内网
    interface: 127.0.0.1
    interface: 10.8.0.1        # 你的内网/VPN 网卡 IP
    access-control: 127.0.0.0/8 allow
    access-control: 10.0.0.0/8 allow
    access-control: ::1 allow

    # 隐私与安全
    do-not-query-localhost: no
    qname-minimisation: yes     # QNAME 最小化，减少隐私泄露
    hide-identity: yes
    hide-version: yes
    prefetch: yes               # 热门域名主动预取
    cache-min-ttl: 300
    cache-max-ttl: 86400

    # DNSSEC 全程开启
    auto-trust-anchor-file: "/var/lib/unbound/root.key"
    val-log-level: 2

    # 本地递归，不走任何第三方上游
    root-hints: "/var/lib/unbound/root.hints"
```

获取根提示并启动：

```bash
curl -o /var/lib/unbound/root.hints https://www.internic.net/domain/named.cache
unbound-checkconf && systemctl enable --now unbound
dig @127.0.0.1 cloudflare.com A +dnssec   # 验证：应显示 ad 标志（authenticated）
```

### 验证 DNSSEC 生效

```bash
dig @127.0.0.1 www.ietf.org A +dnssec | grep -E "status|flags"
# 期望输出包含: flags: qr rd ra ad（ad = 校验通过）
# 故意测一个已知坏签名域名，应返回 SERVFAIL
dig @127.0.0.1 dnssec-failed.org A
```

## 方案 B：AdGuard Home 全家桶（去广告+加速）

AdGuard Home 提供 Web 管理面、规则过滤、每客户端统计，是最易用的入口层。Docker 部署：

```bash
mkdir -p /opt/adguard && cat > /opt/adguard/docker-compose.yml <<'EOF'
services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "784:784/udp"     # DoQ
      - "853:853/tcp"     # DoT
      - "3000:3000/tcp"   # 初始化向导
      - "80:80/tcp"       # Web 管理（改 8080 更稳）
    volumes:
      - /opt/adguard/work:/opt/adguard/work
      - /opt/adguard/conf:/opt/adguard/conf
EOF
cd /opt/adguard && docker compose up -d
```

### 关键设置

1. **初始化**：浏览器访问 `http://VPS_IP:3000`，管理端口填 80，DNS 端口默认 53。
2. **上游 DNS 指向本地 Unbound**：设置 → DNS 设置 → 上游服务器填 `127.0.0.1:5335`（记得把 Unbound 也监听 5335 或改端口）。
3. **过滤规则**：内置「AdGuard DNS filter」+ 自行追加 EasyList China、乘风规则等；开启「安全搜索」。
4. **缓存**：开缓存并设 300s，命中率通常可达 40–60%。
5. **查询日志**：按需保留 24–72h，用于排查但兼顾隐私。

Web 管理面每客户端维度统计，能直观看到哪台设备在疯狂请求哪个域名——排查 IoT 设备「打电话回家」神器。

## 方案 C：PowerDNS 权威托管（自管域名）

自建权威的意义：完全掌控记录、秒级生效、不依赖注册商面板。安装：

```bash
apt install -y pdns-server pdns-backend-sqlite3
```

最小配置 `/etc/powerdns/pdns.conf`：

```ini
launch=gsqlite3
gsqlite3-database=/var/lib/powerdns/pdns.sqlite3
api=yes
api-key=请更换为长随机串
webserver=yes
webserver-address=127.0.0.1
webserver-port=8081
```

创建 zone（以 example.com 为例）：

```bash
pdnsutil create-zone example.com ns1.example.com
pdnsutil add-record example.com '' A 203.0.113.10
pdnsutil add-record example.com www A 203.0.113.10
pdnsutil add-record example.com '' MX '10 mail.example.com.'
pdnsutil add-record example.com _dmarc TXT '"v=DMARC1; p=quarantine"'
pdnsutil check-zone example.com
pdnsutil list-zone example.com
```

去注册商把 `example.com` 的 NS 改为 `ns1.example.com`（胶水记录指向你 VPS 的 IP），即完成托管切换。

## DoH / DoT / DoQ 加密出口配置

自建 DNS 的上游/下游都建议加密：

### 下游（客户端 → 你的服务器）启用 DoT

AdGuard Home 默认监听 853 即 DoT，但需要证书：

```bash
# 用 acme.sh + DNS 验证签发证书（示例：dns_cf 走 Cloudflare API）
curl https://get.acme.sh | sh
~/.acme.sh/acme.sh --issue --dns dns_cf -d dns.example.com
```

然后在 AdGuard 设置中启用 DoT/DoH 并指定证书路径。客户端（如手机私人 DNS）填 `dns.example.com` 即加密接入。

### 上游（你的服务器 → 根之外）选择

纯递归形态没有上游；若仍想走加密转发（如 VPS 本身网络环境受限），Unbound 可配 DoT 转发：

```ini
forward-zone:
    name: "."
    forward-tls-upstream: yes
    forward-addr: 1.1.1.1@853#cloudflare-dns.com
    forward-addr: 8.8.8.8@853#dns.google
```

> 提示：QNAME 最小化 + 本地递归在隐私上优于「加密转发给第三方」；转发只在你需要规避本地递归被干扰时使用。

## 分流策略：国内直连与国外加密

跨境场景（自建用于科学上网环境）推荐「国内域名走国内 DNS、其余走加密/递归」的双轨：

```bash
# dnsmasq 分流示例（也可用 AdGuard 的「DNS 重写」实现）
apt install -y dnsmasq
cat > /etc/dnsmasq.d/split.conf <<'EOF'
# 国内域名直接交给国内公共 DNS（低延迟、符合合规）
server=/cn/223.5.5.5
server=/baidu.com/223.5.5.5
server=/taobao.com/223.5.5.5
# 其余全部走本地递归/加密
server=/#/127.0.0.1#5335
EOF
```

更省心的方案：直接维护一份国内域名列表（如 `china-lite` 规则集），在 AdGuard Home 的「DNS 重写」中按域名批量指定上游。目标是：**国内站解析出国内 IP（快），国外站解析不被污染（准）**。

## 性能调优与缓存策略

```bash
# Unbound 缓存调优参考
server:
    msg-cache-size: 64m
    rrset-cache-size: 128m
    key-cache-size: 64m
    neg-cache-size: 64m
    num-threads: 2            # 与 vCPU 数匹配
    so-rcvbuf: 4m
    so-sndbuf: 4m
    outgoing-range: 4096      # 高并发时提升
    infra-cache-numhosts: 20000
```

经验值：

| 参数 | 默认 | 推荐（2C4G VPS） |
|------|------|------------------|
| rrset-cache-size | 4m | 128m |
| num-threads | 1 | 2 |
| cache-min-ttl | 0 | 300 |
| 缓存命中率 | — | 目标 ≥ 40% |

用 `unbound-control stats` 观察命中率：

```bash
unbound-control stats | grep -E "total.num.queries|cache.hits|cache.miss"
# 命中率 = hits / (hits+miss)
```

## 安全与防泄漏

1. **限制来源**：access-control 只放行自己的 IP 段；公网 VPS 用防火墙白名单 53/853 端口来源。
2. **防开放递归被滥用**：开放递归会被用于 DDoS 放大攻击，云厂商会封端口——务必白名单。
3. **DNSSEC 全程开启**：防缓存投毒（Kaminsky 类攻击）。
4. **系统级防泄漏**：若用于代理环境，确认浏览器/系统没有绕过 DNS 走系统默认（开启「安全 DNS」会直连 DoH，反而绕过你的过滤）。
5. **更新规则**：广告过滤规则每周更新；Unbound/PowerDNS 走 apt 自动安全更新。
6. **日志脱敏**：查询日志定期清理；管理界面绑定内网或加访问密码。

## 客户端接入：路由器与系统设置

### 路由器（推荐，全屋生效）

OpenWrt：网络 → DHCP/DNS → DNS 转发填 `VPS_IP#53`；或直接改 `dnsmasq` 的 `server` 指向。

### 系统级 DoT（手机出门也加密）

- **Android 11+**：设置 → 网络 → 私人 DNS → 填 `dns.example.com`（DoT）。
- **iOS/macOS**：描述文件配置 DoH/DoT，或装 AdGuard 官方 App。
- **Windows**：网络适配器 DNS 手动填 VPS IP（内网/VPN 场景）。

## 一键部署与监控脚本

```bash
#!/usr/bin/env bash
# dns-stack-health.sh —— DNS 体系健康体检
echo "===== 服务状态 ====="
systemctl is-active unbound && echo "unbound OK"
docker ps --format '{{.Names}}: {{.Status}}' | grep adguard
echo "===== 解析连通性 ====="
dig @127.0.0.1 www.baidu.com +short | head -2          # 国内
dig @127.0.0.1 www.github.com +short | head -2         # 国外
echo "===== DNSSEC 校验 ====="
dig @127.0.0.1 www.ietf.org +dnssec | grep "flags:" | grep -q "ad" && echo "DNSSEC OK" || echo "DNSSEC FAIL"
echo "===== Unbound 命中率 ====="
unbound-control stats | awk -F= '/total.num.queries/{q=$2}/cache.hits/{h=$2}END{printf "命中率: %.1f%%\n", h/q*100}'
echo "===== 53 端口暴露检查 ====="
ss -lntup | grep ":53 " || echo "无 53 监听（正常，若仅内网使用）"
```

加入 crontab 每日执行，异常即告警（可对接 Telegram Bot）。

## 常见问题 FAQ

**Q: 自建 DNS 会不会更慢？**
A: 冷缓存时首次递归比公共 DNS 慢几十 ms，但大缓存 + 预取后热点域名通常更快；AdGuard 命中率上去后体感明显变快。

**Q: 53 端口被云厂商封了？**
A: 部分厂商默认封 53 入站。对策：用 DoT 853 / DoQ 784 / DoH 443 端口替代；或工单申请解封。

**Q: 广告过滤误伤正常网站？**
A: 在 AdGuard 查询日志中找到该域名，加入白名单；误报规则可向对应规则源反馈。

**Q: 递归解析偶尔 SERVFAIL？**
A: 多为 DNSSEC 校验失败或根提示过期。先 `unbound-control reload`，再查 `unbound-control log`；确认系统时间准确（NTP）。

**Q: 能提升科学上网速度吗？**
A: 能缓解 DNS 污染导致的连接超时/假 IP；实际速率瓶颈仍在链路本身，DNS 只解决「找对路」。

**Q: 与 AdGuard 公开 DNS / NextDNS 相比呢？**
A: 自建优势是数据不出自己服务器、规则自定义无限制；劣势是要维护高可用（单点故障）。可接受偶尔不可用就自建，追求 99.99% 用商业服务。

**Q: 需要什么配置的 VPS？**
A: 自用 1 核 512M 足够（Unbound+AdGuard 内存合计 < 300M）；家庭多设备+高查询量建议 1G。

## 相关资源与推荐入口

- https://vpsvip.net - VPSVIP 官网（低配实惠型 VPS 情报，适合 DNS 常驻机）
- https://clashvip.net - ClashVIP 官网
- https://nav.clashvip.net - ClashVIP 精选导航
- https://clashhub.net - ClashHub 社区
- https://bbs.clashhub.net - ClashHub 论坛
- https://clash-for-windows.net - Clash for Windows 官方网站
- https://nlnetlabs.nl/projects/unbound - Unbound 官方文档
- https://github.com/AdguardTeam/AdGuardHome - AdGuard Home 项目
- https://doc.powerdns.com - PowerDNS 官方文档

## 免责声明

1. 本仓库内容仅供技术学习与信息参考。
2. 请遵守所在地法律法规；跨境网络行为请自行评估合规风险。
3. 暴露公网服务前务必做好白名单与安全加固。
4. 广告过滤规则来源于第三方社区，请按需选用。

## 许可证

MIT License

---
更新时间：2026-09-09
