# Agent Instructions

This repository documents an IPv6-only Debian VPS with WARP IPv4 and Mihomo selective routing. Preserve the live host's native IPv6 and SSH connectivity.

- Never commit real `mihomo/config.yaml`, `/etc/wireguard/warp.conf`, WARP account data, node UUID, or credentials. Templates must keep `REPLACE_WITH_*` placeholders. Inspect `git diff --cached` before committing.
- Treat WARP as user-managed. Do not uninstall it, replace its generated account configuration, or switch it to dual-stack mode. The target is WARP for ordinary IPv4, native IPv6 for ordinary IPv6, and `hinet` for the Anthropic/Claude and supplemental rules in the Mihomo template.
- Check service state, routes, DNS, and existing SSH sessions before touching network configuration. Do not reboot the machine merely to test startup.
- Validate Mihomo and dnsmasq configuration before service restarts. Start the local DNS service and query it directly before changing the system resolver. Back up `/etc/resolvconf.conf` before editing it.
- Keep the narrow main-table routes for `198.18.0.0/16` and `160.79.104.0/21` through the Mihomo `Meta` interface. WARP's policy rule can otherwise capture these IPv4 destinations.
- Verify both address families: Claude/Anthropic requests must log `using hinet`; ordinary IPv4 must show `warp=on`; ordinary IPv6 must retain the VPS address; SSH must stay connected.
- Report the limits of domain routing: shared third-party domains also route unrelated traffic; keyword rules outside listed DNS suffixes rely on sniffing; IPv4 ASN prefixes outside the explicit route may be captured by WARP; application DoH, hard-coded IPs, and a custom `ANTHROPIC_BASE_URL` outside the listed suffixes are not fully covered.
