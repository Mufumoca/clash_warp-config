# IPv6-only VPS: WARP IPv4 + Mihomo Claude 分流

这套配置用于有原生 IPv6、无原生 IPv4 的 Debian VPS：普通 IPv4 通过 WARP 出站，普通 IPv6 保持原生出口；Anthropic、Claude 及下述补充域名通过 Mihomo 的 `hinet` VLESS Reality 节点。Mihomo 同时提供仅监听本机的 HTTP/SOCKS 混合端口 `127.0.0.1:7890`。

仓库只存放公开模板。**不要提交** `/etc/wireguard/warp.conf`、WARP 账户文件、填入节点参数后的 `config.yaml`、缓存、日志或任何密钥。

## 设计

启动顺序是 `mihomo-dns` → `wg-quick@warp` → `mihomo`。`dnsmasq` 只把列出的域名和常见 Datadog、Sift 后缀交给 Mihomo DNS；其他域名使用独立的 IPv6 上游 DNS。因此 WARP 端点不会被解析为 fake-IP，Mihomo 停止时普通域名仍可解析。

匹配域名得到 `198.18.0.0/16` 或 `fdfe:dcba:9876::/64` 中的 fake-IP。IPv4 fake-IP 的主路由必须指向 Mihomo TUN：WARP 重连后，其策略路由可能排在 Mihomo 前面，单靠 `tun.auto-route` 会让 IPv4 fake-IP 错误地进入 WARP。`mihomo-fakeip-route` 在 Mihomo 每次启动时为 fake-IP 和补充的 `160.79.104.0/21` 各添加一条窄范围路由。Mihomo 对匹配项只选 `hinet`，没有 `DIRECT` 备用节点；不匹配的流量走 `DIRECT`，沿主机当前出口分别使用 WARP IPv4 或原生 IPv6。`IP-ASN` 规则首次使用时会下载 ASN 数据库。

域名规则涵盖 `anthropic.com`、`claude.ai`、`claude.com`、`clau.de`、`claudemcpclient.com`、`claudemcpcontent.com`、`claudeusercontent.com`，以及模板中列出的 CDN、认证、内容、遥测和客服域名。`sentry.io`、`statsigapi.net`、`intercom.io`、`intercomcdn.com` 与 `datadog`、`sift` 关键词属于共享服务；这些域名上的其他应用流量也会走 `hinet`。

## 前提

- Debian 13、root 权限、可用的 `/dev/net/tun` 和原生 IPv6；本仓库的路径与服务单元按 root 部署。
- `openresolv` 提供 `resolvconf`。若系统使用 `systemd-resolved` 或其他 DNS 管理器，需要按该管理器调整本机 DNS；不要直接覆盖其配置。
- 保持一个可用的 SSH 会话。以下步骤不要求重启 VPS，也不要在未验证解析前修改系统 DNS。

## 1. 安装 WARP IPv4

使用 [fscarmen/warp](https://gitlab.com/fscarmen/warp) 脚本安装 IPv4 模式。先检查下载的脚本，再运行：

```bash
curl -fL https://gitlab.com/fscarmen/warp/-/raw/main/menu.sh -o menu.sh
bash menu.sh 4
```

选择 WireGuard 内核和全局 IPv4 模式。脚本应生成 `/etc/wireguard/warp.conf`、保活脚本及 `wg-quick@warp.service`。`warp/warp.conf.example` 仅供对照，**不要直接替换**脚本生成的带账户信息的配置。确认：

```bash
systemctl is-active wg-quick@warp.service
systemctl is-enabled wg-quick@warp.service
wg show warp endpoints
curl -4 -sS https://www.cloudflare.com/cdn-cgi/trace
```

IPv4 trace 应有 `warp=on`。IPv6-only VPS 的 WARP 对端应解析为可直连的 IPv6 地址。保留脚本为原生 IPv6 源网段生成的 `lookup main` 规则，以免影响 SSH 等入站连接。

## 2. 安装 Mihomo

下面固定了本机验证过的 `v1.19.31` Linux amd64 发布包及 SHA-256；其他架构请在 [官方发布页](https://github.com/MetaCubeX/mihomo/releases)选择对应文件和校验值。

```bash
curl -fL https://github.com/MetaCubeX/mihomo/releases/download/v1.19.31/mihomo-linux-amd64-v1-v1.19.31.gz -o /tmp/mihomo.gz
printf '%s  %s\n' d4304c546c3cddcb6fafd4b4fddb0ba1a95ffa36606fda56d75db2e59ad24114 /tmp/mihomo.gz | sha256sum -c -
gzip -dc /tmp/mihomo.gz > /tmp/mihomo
install -m 0755 /tmp/mihomo /usr/local/bin/mihomo
apt-get install -y dnsmasq-base
```

复制模板到本机私有目录，填写 `REPLACE_WITH_*`：节点 IPv6、端口、UUID、Reality SNI、公钥、short ID，以及 `wg show warp endpoints` 中的 WARP 对端 IPv6。两个 `route-exclude-address` 均使用 `/128`。不要把填好的文件复制回仓库。

```bash
install -d -m 0700 /root/.config/mihomo
install -m 0600 mihomo/config.yaml.example /root/.config/mihomo/config.yaml
install -m 0644 dns/dnsmasq.conf /root/.config/mihomo/dnsmasq.conf
install -D -m 0755 scripts/mihomo-fakeip-route /usr/local/libexec/mihomo-fakeip-route
```

填写后先校验配置：

```bash
/usr/local/bin/mihomo -t -d /root/.config/mihomo -f /root/.config/mihomo/config.yaml
dnsmasq --test --conf-file=/root/.config/mihomo/dnsmasq.conf
```

## 3. 安装服务和 DNS

```bash
install -m 0644 systemd/mihomo-dns.service /etc/systemd/system/mihomo-dns.service
install -m 0644 systemd/mihomo.service /etc/systemd/system/mihomo.service
install -d -m 0755 /etc/systemd/system/wg-quick@warp.service.d
install -m 0644 systemd/10-local-dns.conf /etc/systemd/system/wg-quick@warp.service.d/10-local-dns.conf
systemctl daemon-reload
systemctl enable --now mihomo-dns.service wg-quick@warp.service mihomo.service
```

**先不修改系统解析器**，查询本地 DNS：

```bash
dig @127.0.0.1 +short api.anthropic.com A
dig @127.0.0.1 +short claude.ai AAAA
dig @127.0.0.1 +short example.com A
dig @127.0.0.1 +short engage.cloudflareclient.com AAAA
```

前两项应为 fake-IP；后两项应是真实公网地址。通过后先备份 `/etc/resolvconf.conf`：

```bash
cp -a /etc/resolvconf.conf /etc/resolvconf.conf.before-clash-warp
```

再参照 `resolvconf/resolvconf.conf.example` 将 `name_servers=127.0.0.1` **合并**到现有 `/etc/resolvconf.conf`，不要清除其他自定义设置。最后更新并检查系统解析器：

```bash
resolvconf -u
cat /etc/resolv.conf
```

`/etc/resolv.conf` 应只有 `nameserver 127.0.0.1`。`resolvconf -u` 也可用来确认 WARP 更新 DNS 后，本机设置仍然保留。

## 4. 验证

```bash
systemctl is-active mihomo-dns wg-quick@warp mihomo
systemctl is-enabled mihomo-dns wg-quick@warp mihomo
getent ahostsv4 api.anthropic.com
ip -4 route get 198.18.0.4
ip -4 route get 160.79.104.1
curl -4 -sS -o /dev/null -w '%{http_code}\n' https://api.anthropic.com/
curl -6 -sS -o /dev/null -w '%{http_code}\n' https://claude.ai/
journalctl -u mihomo -n 100 --no-pager | grep -E 'api.anthropic.com|claude.ai'
curl -4 -sS https://www.cloudflare.com/cdn-cgi/trace
curl -6 -sS https://www.cloudflare.com/cdn-cgi/trace
```

`getent` 应返回 fake-IP；该地址和 `160.79.104.1` 都应由 `Meta` TUN 路由。日志中的 Anthropic / Claude 行必须显示 `using hinet`。Cloudflare trace 对普通 IPv4 应显示 `warp=on`，对原生 IPv6 应显示 VPS 的 IPv6 且 `warp=off`。`claude.ai` 可能返回网站的 `403`，应以 Mihomo 命中日志判断出站规则，而不是把 HTTP 状态码当成出口证明。

如果 DNS 切换失败，先把备份的 `/etc/resolvconf.conf.before-clash-warp` 恢复到 `/etc/resolvconf.conf` 并执行 `resolvconf -u`；这不需要停止 WARP。不要在只有一条 SSH 连接时直接重启整机测试。

## 边界

规则匹配的是列出的域名、IP 段和 ASN。`dnsmasq` 无法按任意子串匹配 DNS 查询，因此 `DOMAIN-KEYWORD,datadog` 和 `DOMAIN-KEYWORD,sift` 中未列入 DNS 配置的域名依赖 Mihomo 的 TUN/SNI 嗅探，不能保证所有协议都命中。IPv4 ASN 中除 `160.79.104.0/21` 外的地址仍可能先被 WARP 策略路由捕获。应用内 DoH、硬编码目标 IP，或者自定义 `ANTHROPIC_BASE_URL` 指向其他域名，也可能绕过域名规则。不要把共享的 `github.com`、`storage.googleapis.com` 或 `registry.npmjs.org` 全域名加入 `hinet`，除非明确希望这些服务的全部流量也走该节点。
