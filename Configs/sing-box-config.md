---
tags:
  - vpn
  - xray
  - config
---

{
    "dns": {
        "independent_cache": true,
        "servers": [{
                "address": "https://8.8.8.8/dns-query",
                "detour": "proxy",
                "strategy": "",
                "tag": "dns-remote"
            },
            {
                "tag": "local-dns",
                "address": "8.8.8.8",
                "detour": "direct-out"
            }],
        "rules": [{
            "domain": "gospodinnikto.duckdns.org",
            "server": "local-dns"
        }]
    },
    "inbounds": [{
            "auto_route": true,
            "inet4_route_exclude_address": [
                "192.168.0.0/16"
            ],
            "domain_strategy": "",
            "endpoint_independent_nat": true,
            "inet4_address": "172.19.0.1/28",
            "interface_name": "singbox",
            "mtu": 9000,
            "sniff": true,
            "sniff_override_destination": false,
            "stack": "gvisor",
            "strict_route": true,
            "tag": "tun-in",
            "type": "tun"
        }
    ],
    "log": {
        "level": "info"
    },
    "outbounds": [{
            "domain_strategy": "",
            "flow": "xtls-rprx-vision",
            "packet_encoding": "",
            "server": "5.35.70.91",
            "server_port": 443,
            "tag": "proxy",
            "tls": {
                "enabled": true,
                "reality": {
                    "enabled": true,
                    "public_key": "x04eoQz0sX8NNotaYg_jXh147tq5zT59GxKKiIMwDy0",
                    "short_id": "f5"
                },
                "server_name": "www.samsung.com",
                "utls": {
                    "enabled": true,
                    "fingerprint": "random"
                }
            },
            "type": "vless",
            "uuid": "62b457ec-992c-44dc-a98c-aa6317e0e681"
        }, 
        {
            "tag": "direct-out",
            "type": "direct"
        },  
        {
            "tag": "block",
            "type": "block"
        }, 
        {
            "tag": "dns-out",
            "type": "dns"
        }
    ],
    "route": {
        "auto_detect_interface": true,
        "final": "proxy",
        "rules": [{
                "outbound": "dns-out",
                "protocol": "dns"
            }
        ]
    }
}

## Related Notes
- [[Mikrotik + SingBox с VLess]]
