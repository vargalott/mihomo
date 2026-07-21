## mihomo

A proxy setup using [mihomo](https://wiki.metacubex.one/en/) designed for a specific censorship-circumvention use case

```
git clone https://github.com/vargalott/mihomo
cd mihomo
mkdir data && cp config.example.yaml data/config.yaml
# edit config.yaml
docker compose up -d
```

<img src="https://raw.githubusercontent.com/MetaCubeX/metacubexd/refs/heads/main/docs/pc/overview.png"/>

---

### Design

#### Dual-gateway

| | |
|-|-|
| `PROXY` | Selects which proxy protocol carries final non-RU traffic |
| `GATEWAY-PROXY` | Controls how `PROXY`-bound traffic (`VLESS`/`HYSTERIA`/etc) exits - i.e., what the proxy client itself routes through |
| `GATEWAY-RU` | Controls how `RU` traffic exits independently |

Main proxies carries `dialer-proxy: GATEWAY-PROXY` - this means the proxy client connections themselves are subject to a second routing decision; for example, you can route main proxy through `WHITELIST-CIDR-GEORU` or `WHITELIST-BYPASS` (a split-tunnel whitelist of RU CIDRs) instead of `DIRECT` so that the proxy server appears reachable even if the local ISP/DPI applies selective blocking

```
traffic
   |
   v
mihomo (tun/mixed)
   |
   |--> IPv6 -------------------------> REJECT
   |--> UDP/443 (quic) ---------------> REJECT
   |--> private ranges ---------------> DIRECT
   |
   |--> specific rules(proxy) --------> PROXY ----------> dialer-proxy: GATEWAY-PROXY --> DIRECT | WHITELIST-CIDR-GEORU | WHITELIST-BYPASS
   |
   |--> RU domains/ips ---------------> GATEWAY-RU -----> DIRECT | WHITELIST-CIDR-GEORU | WHITELIST-BYPASS
   |
   |--> everything else --------------> PROXY ----------> dialer-proxy: GATEWAY-PROXY --> DIRECT | WHITELIST-CIDR-GEORU | WHITELIST-BYPASS
```

#### Ingress

Traffic enters Mihomo two ways:
- `tun interface` (gvisor stack) - captures all TCP/UDP system-wide without default per-app configuration; `auto-route`, `auto-redirect`, and `strict-route` are all enabled; captured traffic from `tun` with `find-process-mode: always` can be routed with additional rules
- `mixed listener` on `127.0.0.1:8888` - HTTP/Socks5 proxy with username/password auth, for apps that prefer explicit proxy configuration or in case when tun is disabled

#### Example protocols

| | |
|-|-|
| VLESS + TCP(XTLS Vision) + REALITY | high-performance; censorship-resistant; client-fingerprint spoofing (chrome) |
| VLESS + XHTTP + REALITY | multiplexed transport with stream-one mode and connection reuse via xmux |
| Hysteria2 (QUIC-based) | fast for high-bandwidth; password auth |

---

### DNS

#### Concept

| | | |
|-|-|-|
| `.lan` + `localhost` | passthrough | real-ip |
| `stun`/`turn` | Cloudflare DoH/DoT | real-ip |
| RU services (all `geosite-*-ru` + yandex sets + additional) | Yandex DNS (`77.88.8.8`, DoH/DoT) | real-ip |
| Any other specified services | Cloudflare DoH/DoT | real-ip |
| Everything else | upstream proxy server resolver | fake-ip |

`fake-ip` mode returns synthetic address from `198.18.0.0/15` for all non-filtered domains; mihomo intercepts connections to those address and routes them based on rules, without ever resolving the real ip on the client; this prevents DNS leaks for proxied traffic

`real-ip` mode is used for the specified cases so they resolve to actual address, enabling ip-based rules to function correctly and allowing direct or whitelisted routing

#### Nameservers

- `default-nameserver`: Cloudflare (`1.1.1.1`/`1.0.0.1`) - used to bootstrap DoH/DoT resolver addresses
- `nameserver`: Cloudflare DoH + DoT - resolver for real-ip requests that doesn't fall under `nameserver-policy`
- `nameserver-policy`: `.lan` or RU rulesets - router resolver or Yandex DoH/DoT/plain accordingly

---

### Rules

#### Routing table

These rules can be easily extended and adjusted if necessary
Telegram in this example is explicitly matched before the RU sets to ensure it always goes through the proxy even if it has RU-associated address
```
IP-CIDR6 ::/0                     REJECT           # kill all IPv6
UDP port 443                      REJECT           # kill QUIC (force TCP for TLS)
geoip-private                     DIRECT           # LAN/loopback bypass

geosite-telegram                  PROXY
geoip-telegram                    PROXY

geosite-yandex                    GATEWAY-RU
geosite-bank-ru                   GATEWAY-RU
geosite-ru-available-only-inside  GATEWAY-RU
geosite-gov-ru                    GATEWAY-RU
geosite-category-ru               GATEWAY-RU
geosite-tld-ru                    GATEWAY-RU
geoip-ru                          GATEWAY-RU

MATCH                             PROXY            # default: proxy everything
```

#### Providers

All fetched through `PROXY`, cached locally, `interval: 86400` (daily refresh)

| | | |
|-|-|-|
| `geoip-private` | IP CIDR | [official meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat/tree/meta) |
| `geoip-ru` | IP CIDR | [official meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat/tree/meta) |
| `geoip-telegram` | IP CIDR | [Telegram official CIDR list](https://core.telegram.org/resources/cidr.txt) |
| `geosite-telegram` | Domain | [official meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat/tree/meta) |
| `geosite-category-ru` | Domain | [official meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat/tree/meta) |
| `geosite-tld-ru` | Domain | [official meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat/tree/meta) |
| `geosite-gov-ru` | Domain | [official meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat/tree/meta) |
| `geosite-bank-ru` | Domain | [official meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat/tree/meta) |
| `geosite-yandex` | Domain | [official meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat/tree/meta) |
| `geosite-ru-available-only-inside` | Domain | [itdoginfo/allow-domains](https://github.com/itdoginfo/allow-domains) |

---

### Notes

- Don't forget to change the placeholder values, such as `secret`, `username`, `password`, `SERVER`, `UUID`, `PBK`, `SID`
- `listeners:` block - defines per-listener mixed-in auth, but most GUI clients only read the global `mixed-port` / `authentication` settings (or even sets them unconditionally) and silently ignore listener-scoped config; if auth isn't being enforced, fall back to the commented-out global equivalents in the config
- It is a good design practice to make `proxies: - REJECT` for each `proxy-group` that come from `proxy-provider`; serves as a kill-switch in cases of broken upstream subscription
- `REGULAR-OPENRAY`, `WHITELIST-CIDR-GEORU`, `WHITELIST-BYPASS` are just an examples of importing generic proxy subscriptions(plain text or base64 `vless://`, `ss://`, etc remote configs)
- `ntp`: Cloudflare time server, synced every 30 minutes
- `sniffer`: `HTTP (80, 8080–8880)`, `TLS (443, 8443)`, `QUIC (443, 8443)` - enables domain sniffing from raw TCP/UDP for accurate rule matching without DNS QUIC is still sniffing even tough it's routing is blocked in rules
- `external-controller`: `127.0.0.1:8890` - for dashboards (metacubexd, yacd etc)
- `process-mode`: `always` - enables per-process rule matching if needed
- `tcp-concurrent`: `enabled` - parallel connection attempts for lower latency
