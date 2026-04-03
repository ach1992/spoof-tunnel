# Spoof Tunnel

> **For authorized lab, research, and network-owner testing only.**
>
> This document is a cleaned-up and expanded English README for the project. It focuses on installation, configuration, service management, logging, and troubleshooting on **Ubuntu/Debian**. It intentionally avoids guidance for using the software against networks or policies you do not own or administer.

[فارسی / Persian](README-fa.md)

## What this project is

Spoof Tunnel is a Layer 3 / Layer 4 tunneling proxy that uses raw packets, encrypted transport, and a client/server model. The repository README explains the design goals, the mutual spoofing model, the reliability layer, multiplexing, and the cryptography used by the tunnel. The implementation is written in Go and includes separate client and server modes in a single binary. citeturn197365view0turn526439view2turn983978view0

## What this guide adds

The original README explains the architecture, but it does not fully walk a new user through:

- installing dependencies on Ubuntu/Debian
- building the binary correctly
- generating keys with the actual CLI flags used by the code
- preparing client and server JSON configuration files
- starting the program correctly
- giving the binary the privileges raw sockets require
- running it with `systemd`
- checking logs and troubleshooting startup problems

This file fills those gaps.

---

## Important notes before you start

1. **Use only on systems and networks you own or are explicitly authorized to test.**
2. **Raw sockets require elevated privileges.** The code warns that running without root may fail. citeturn526439view2
3. **A valid config file is mandatory.** The code validates mode, transport, addresses, spoof fields, crypto keys, and other settings before startup. citeturn983978view0
4. **Client and server do not use separate subcommands.** The operational mode comes from the JSON config field `mode`, and the binary is started with `-config`. The current repository README examples showing `server -c ...` / `client -c ...` do not match `main.go`. citeturn526439view2turn983978view0
5. **Key generation uses `-generate-keys`.** The current README says `generate-keys` as a command, but the actual code uses a boolean flag. citeturn526439view2

---

## Tested target environment

This guide is written for:

- Ubuntu 20.04+
- Ubuntu 22.04+
- Ubuntu 24.04+
- Debian 11+
- Debian 12+

You need a 64-bit Linux host with:

- root access or the ability to grant Linux capabilities
- Go installed if you want to build from source
- `systemd` if you want to run it as a service

---

## Repository layout

Important files in the repository:

- `cmd/spoof/main.go` — main program entrypoint and CLI flags
- `config.json.example` — example client config
- `server-config.json.example` — example server config
- `README.md` — English README
- `README-fa.md` — Persian README citeturn793958view0turn526439view0turn526439view1

---

## 1) Install prerequisites on Ubuntu / Debian

Update package lists and install the common tools you will typically want during build and operation:

```bash
sudo apt update
sudo apt install -y git curl wget ca-certificates build-essential pkg-config \
  golang-go libpcap-dev tcpdump jq systemd
```

Notes:

- The project imports `gopacket` and `pcap` in the implementation path described by the README. citeturn230281view0turn197365view0
- If you already have a newer Go toolchain installed manually, you can skip `golang-go`.

Check versions:

```bash
go version
uname -a
```

---

## 2) Download the source

```bash
git clone https://github.com/ParsaKSH/spoof-tunnel.git
cd spoof-tunnel
```

If you prefer to build a tagged release, check the releases page and checkout that tag before building. The repository shows a latest release entry on GitHub. citeturn197365view0

---

## 3) Build the binary

Build for Linux AMD64:

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o spoof ./cmd/spoof/
```

That build command matches the README. citeturn197365view0turn599798view0

Optional: install into `/usr/local/bin`:

```bash
sudo install -m 0755 spoof /usr/local/bin/spoof
```

Check that it starts:

```bash
spoof -version
```

The binary exposes a `-version` flag in `main.go`. citeturn526439view2

---

## 4) Generate keys correctly

Generate a keypair on each side:

```bash
./spoof -generate-keys
```

The code prints:

- a **Private Key** for the local machine
- a **Public Key** to share with the peer

You need:

- the **local private key** in your own config
- the **peer public key** in your own config

This behavior is defined directly in `main.go` and in config validation. citeturn526439view2turn983978view0

Suggested workflow:

1. Generate keys on the server.
2. Generate keys on the client.
3. Exchange only the public keys.
4. Keep each private key only on its own host.

---

## 5) Create directories for configs and logs

```bash
sudo mkdir -p /etc/spoof-tunnel
sudo mkdir -p /var/log/spoof-tunnel
sudo chmod 700 /etc/spoof-tunnel
```

Recommended file names:

- server: `/etc/spoof-tunnel/server.json`
- client: `/etc/spoof-tunnel/client.json`

---

## 6) Understand how startup mode works

The binary does **not** switch modes using subcommands. It reads the `mode` field from the JSON config:

- `"mode": "server"`
- `"mode": "client"` citeturn983978view0

So the correct startup pattern is:

```bash
sudo ./spoof -config /path/to/config.json
```

or, if installed globally:

```bash
sudo spoof -config /path/to/config.json
```

---

## 7) Client configuration example

The repository ships a compact example for client mode. Below is a clearer expanded version with comments removed so it remains valid JSON.

```json
{
  "mode": "client",
  "transport": {
    "type": "udp",
    "icmp_mode": "echo",
    "protocol_number": 0
  },
  "listen": {
    "address": "127.0.0.1",
    "port": 1080
  },
  "server": {
    "address": "SERVER_REAL_IP",
    "port": 8080
  },
  "spoof": {
    "source_ip": "CLIENT_SPOOF_IP",
    "peer_spoof_ip": "SERVER_SPOOF_IP"
  },
  "crypto": {
    "private_key": "CLIENT_PRIVATE_KEY_BASE64",
    "peer_public_key": "SERVER_PUBLIC_KEY_BASE64"
  },
  "performance": {
    "buffer_size": 65535,
    "mtu": 1400,
    "session_timeout": 600,
    "workers": 4,
    "read_buffer": 4194304,
    "write_buffer": 4194304,
    "send_rate_limit": 1000
  },
  "reliability": {
    "enabled": true,
    "window_size": 128,
    "retransmit_timeout_ms": 300,
    "max_retries": 5,
    "ack_interval_ms": 50
  },
  "fec": {
    "enabled": false,
    "data_shards": 10,
    "parity_shards": 3
  },
  "keepalive": {
    "enabled": true,
    "interval_seconds": 30,
    "timeout_seconds": 120
  },
  "logging": {
    "level": "info",
    "file": "/var/log/spoof-tunnel/client.log"
  }
}
```

Why this differs from `config.json.example`:

- the shipped example is very compressed and omits some fields that the code supports by default
- `send_rate_limit` exists in `PerformanceConfig` even though it is not documented in the current README table. citeturn526439view0turn983978view0

### Client field meanings

- `listen.address` / `listen.port`: where the local SOCKS5 proxy will bind. Default is `127.0.0.1:1080` if not set. citeturn983978view0
- `server.address` / `server.port`: the actual server endpoint the client sends packets toward.
- `spoof.source_ip`: the local side’s chosen spoof identity.
- `spoof.peer_spoof_ip`: the expected source IP of packets from the peer, used by the filter logic.
- `crypto.private_key`: this client’s private key.
- `crypto.peer_public_key`: the server’s public key.

---

## 8) Server configuration example

The repository includes `server-config.json.example`. Below is a cleaned-up version suitable as a starting point.

```json
{
  "mode": "server",
  "transport": {
    "type": "icmp",
    "icmp_mode": "echo",
    "protocol_number": 0
  },
  "listen": {
    "address": "0.0.0.0",
    "port": 8080
  },
  "spoof": {
    "source_ip": "SERVER_SPOOF_IP",
    "source_ipv6": "",
    "peer_spoof_ip": "CLIENT_SPOOF_IP",
    "peer_spoof_ipv6": "",
    "client_real_ip": "CLIENT_REAL_IP",
    "client_real_ipv6": ""
  },
  "crypto": {
    "private_key": "SERVER_PRIVATE_KEY_BASE64",
    "peer_public_key": "CLIENT_PUBLIC_KEY_BASE64"
  },
  "performance": {
    "buffer_size": 131072,
    "mtu": 1400,
    "session_timeout": 600,
    "workers": 16,
    "read_buffer": 16777216,
    "write_buffer": 16777216,
    "send_rate_limit": 1000
  },
  "reliability": {
    "enabled": true,
    "window_size": 128,
    "retransmit_timeout_ms": 300,
    "max_retries": 5,
    "ack_interval_ms": 50
  },
  "fec": {
    "enabled": true,
    "data_shards": 10,
    "parity_shards": 3
  },
  "keepalive": {
    "enabled": true,
    "interval_seconds": 30,
    "timeout_seconds": 120
  },
  "logging": {
    "level": "info",
    "file": "/var/log/spoof-tunnel/server.log"
  }
}
```

### Server-specific requirements

In server mode, config validation requires `client_real_ip` or `client_real_ipv6`. Without it, startup will fail. citeturn983978view0

---

## 9) Full configuration reference

### Required fields

#### Required on both client and server

- `mode`
- `transport.type`
- at least one spoof source address: `spoof.source_ip` or `spoof.source_ipv6`
- `crypto.private_key`
- `crypto.peer_public_key` citeturn983978view0

#### Required in client mode

- `server.address`
- `server.port` citeturn983978view0

#### Required in server mode

- `spoof.client_real_ip` or `spoof.client_real_ipv6` citeturn983978view0

### Transport section

- `transport.type`: `udp`, `icmp`, or `raw` are accepted by the config validator. The current README mainly documents `udp` and `icmp`. citeturn983978view0turn197365view0
- `transport.icmp_mode`: `echo` or `reply` when using ICMP.
- `transport.protocol_number`: used only if `type` is `raw`, must be 1–255. citeturn983978view0

### Listen section

- `listen.address`: IP address to bind locally.
- `listen.port`: bind port.
- defaults: `127.0.0.1:1080` in client mode, `127.0.0.1:8080` unless overridden, though for a real server you will usually set `0.0.0.0`. Defaults are applied in code. citeturn983978view0

### Performance section

- `buffer_size`: packet buffer size
- `mtu`: tunnel payload size before encapsulation; lower it if fragmentation appears
- `session_timeout`: inactivity timeout in seconds
- `workers`: number of packet workers
- `read_buffer` / `write_buffer`: socket buffer sizes
- `send_rate_limit`: packets per second; field exists in code even though the examples do not show it clearly. citeturn983978view0

### Reliability section

Defaults are automatically applied for window size, retransmit timeout, retry count, and ACK interval if omitted. citeturn983978view0

### FEC section

If `fec.enabled` is true:

- `data_shards >= 1`
- `parity_shards >= 1`
- `data_shards + parity_shards <= 256` citeturn983978view0

### Keepalive section

Default values are applied in code for interval and timeout if omitted. citeturn983978view0

---

## 10) Run it manually first

### Server

```bash
sudo spoof -config /etc/spoof-tunnel/server.json
```

### Client

```bash
sudo spoof -config /etc/spoof-tunnel/client.json
```

Expected behavior:

- server logs startup, transport type, and spoof source IP
- client logs startup, server destination, spoof source IP, and the local SOCKS5 bind address
- on success, the client exposes a SOCKS5 proxy on the configured `listen.address:listen.port` (commonly `127.0.0.1:1080`) citeturn526439view2

---

## 11) Run without full root using capabilities

If you do not want to run the binary via `sudo`, grant the necessary capability:

```bash
sudo setcap cap_net_raw+ep /usr/local/bin/spoof
getcap /usr/local/bin/spoof
```

Possible output:

```bash
/usr/local/bin/spoof cap_net_raw=ep
```

If packet capture behavior still needs broader privileges on your host, root may still be simpler.

---

## 12) Example `systemd` unit files

### Server service

Create `/etc/systemd/system/spoof-tunnel-server.service`:

```ini
[Unit]
Description=Spoof Tunnel Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/spoof -config /etc/spoof-tunnel/server.json
Restart=always
RestartSec=3
User=root
AmbientCapabilities=CAP_NET_RAW
CapabilityBoundingSet=CAP_NET_RAW
NoNewPrivileges=true
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

### Client service

Create `/etc/systemd/system/spoof-tunnel-client.service`:

```ini
[Unit]
Description=Spoof Tunnel Client
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/spoof -config /etc/spoof-tunnel/client.json
Restart=always
RestartSec=3
User=root
AmbientCapabilities=CAP_NET_RAW
CapabilityBoundingSet=CAP_NET_RAW
NoNewPrivileges=true
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now spoof-tunnel-server
# or
sudo systemctl enable --now spoof-tunnel-client
```

Check status:

```bash
systemctl status spoof-tunnel-server
journalctl -u spoof-tunnel-server -f
```

---

## 13) Using the local SOCKS5 proxy on the client

After the client starts successfully, applications can use the configured local SOCKS5 listener.

Example with `curl`:

```bash
curl --socks5-hostname 127.0.0.1:1080 https://example.com/
```

If you changed `listen.address` or `listen.port`, update the SOCKS endpoint accordingly.

---

## 14) Logging

The code supports:

- `debug`
- `info`
- `warn`
- `error` citeturn983978view0

To log to a file:

```json
"logging": {
  "level": "debug",
  "file": "/var/log/spoof-tunnel/client.log"
}
```

To keep logging on stdout/stderr, leave `file` empty.

Useful commands:

```bash
tail -f /var/log/spoof-tunnel/client.log
tail -f /var/log/spoof-tunnel/server.log
journalctl -u spoof-tunnel-client -f
```

---

## 15) Troubleshooting

### The binary exits immediately with a config error

The config validator is strict. Common causes include:

- invalid `mode`
- missing `server.address` in client mode
- missing `crypto.private_key`
- missing `crypto.peer_public_key`
- missing `spoof.client_real_ip` in server mode
- invalid IP syntax in any spoof or listen fields citeturn983978view0

### “Running without root privileges. Raw sockets may fail.”

This warning comes from the code when EUID is not 0. Run as root or grant `CAP_NET_RAW`. citeturn526439view2

### The client starts but no local proxy appears

Verify:

- `mode` is `client`
- `listen.address` and `listen.port` are valid
- the process did not abort after logging a startup message

### Key errors on startup

If private/public key parsing fails, re-run:

```bash
./spoof -generate-keys
```

and make sure each side uses:

- its **own private key**
- the **other side’s public key**

### MTU-related instability

If transfers seem unstable, lower `performance.mtu`, for example from `1400` to `1300`, then test again. The existing README also notes MTU tuning to avoid fragmentation. citeturn197365view0

### FEC validation errors

If FEC is enabled, make sure shard counts are positive and the total does not exceed 256. citeturn983978view0

---

## 16) Operational checklist

### Server checklist

- [ ] Go and required packages installed
- [ ] binary built or installed
- [ ] server private key generated
- [ ] client public key inserted into `peer_public_key`
- [ ] `mode` set to `server`
- [ ] `spoof.client_real_ip` set
- [ ] logging path writable
- [ ] started with root or `CAP_NET_RAW`

### Client checklist

- [ ] binary built or installed
- [ ] client private key generated
- [ ] server public key inserted into `peer_public_key`
- [ ] `mode` set to `client`
- [ ] `server.address` and `server.port` set
- [ ] local SOCKS5 bind configured
- [ ] started with root or `CAP_NET_RAW`

---

## 17) Known documentation issues fixed in this version

This rewritten README corrects or clarifies the following points from the current repository documentation:

1. **Correct CLI syntax**: use `-config`, not `-c`. The code defines `configPath = flag.String("config", ...)`. citeturn526439view2
2. **Correct key generation syntax**: use `-generate-keys`, not `generate-keys` as a subcommand. citeturn526439view2
3. **Mode comes from config**: no separate `server` or `client` CLI subcommands are implemented. citeturn526439view2turn983978view0
4. **Server requires `client_real_ip`**: this is enforced by validation and should be documented clearly. citeturn983978view0
5. **`raw` transport exists in config validation** even though it is not fully covered in the README tables. citeturn983978view0
6. **`send_rate_limit` exists in code** and is worth documenting for completeness. citeturn983978view0

---

## 18) License

The repository is published under the Apache-2.0 license according to GitHub. citeturn793958view0

---

## 19) Suggested repository file names

If you want to replace the existing docs in your fork, a clean approach is:

- `README.md` → this English file
- `README-fa.md` → the Persian file included alongside it

If you prefer to keep the original files and review first, upload them as:

- `README.en.complete.md`
- `README-fa.complete.md`

