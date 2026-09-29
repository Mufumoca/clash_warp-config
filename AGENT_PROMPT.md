# 给部署 Agent 的提示词

```text
请在我的 Debian IPv6-only VPS 上按本仓库部署或检查 WARP IPv4 + Mihomo 分流。

目标：普通 IPv4 出站走现有 WARP，普通 IPv6 走 VPS 原生 IPv6；anthropic.com、claude.ai、claude.com、claudeusercontent.com 及子域名只走我私下提供的 hinet VLESS Reality 节点。匹配域名不要回落到 DIRECT。三个服务 mihomo-dns、wg-quick@warp、mihomo 都需开机启动，顺序为 DNS → WARP → Mihomo。

先读取 README.md、AGENTS.md，并检查现有服务、/etc/resolv.conf、路由、WireGuard 端点和 SSH。不要卸载或重装已正常工作的 WARP，不要切换到 WARP 双栈，也不要在单一 SSH 连接上重启机器。先测试本地 DNS，再改系统解析器；任何修改都保留可恢复的备份。

节点参数仅写入本机权限为 0600 的 /root/.config/mihomo/config.yaml。不要把 UUID、WARP 私钥、账户资料、token、真实配置或缓存提交到 Git 仓库，也不要把这些值打印在最终报告中。

完成后用真实请求和日志分别验证：Anthropic/Claude 的 IPv4、IPv6 均显示 using hinet；普通 IPv4 的 Cloudflare trace 显示 warp=on；普通 IPv6 显示 VPS 原生地址；SSH 仍可用。说明未覆盖应用内 DoH、硬编码 IP 和自定义 ANTHROPIC_BASE_URL 的域名。
```
