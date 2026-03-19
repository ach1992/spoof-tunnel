# Spoof Tunnel

> **فقط برای محیط‌های آزمایشگاهی، پژوهشی و تست روی شبکه‌هایی که مالک آن هستید یا مجوز صریح دارید.**
>
> این راهنما روی **Ubuntu / Debian** تمرکز دارد و شامل نصب از سورس، نصب دستی از روی باینری، ساخت سرویس `systemd`، ساخت کانفیگ، اجرای برنامه، لاگ‌گیری و عیب‌یابی است.

[English](README.md)

---

## این پروژه چیست؟

Spoof Tunnel یک ابزار client/server مبتنی بر Go است که در یک باینری واحد، هر دو حالت **client** و **server** را پشتیبانی می‌کند. حالت اجرا از داخل فایل config و با فیلد `mode` مشخص می‌شود و برنامه با فلگ `-config` اجرا می‌شود.

---

## سیستم‌عامل هدف

- Ubuntu 20.04+
- Ubuntu 22.04+
- Ubuntu 24.04+
- Debian 11+
- Debian 12+

نیازمندی‌ها:

- Linux 64-bit
- دسترسی `root` یا امکان دادن capability به باینری
- `systemd` برای سرویس دائمی

---

## ساختار مهم مخزن

- `cmd/spoof/main.go` — نقطهٔ ورود برنامه و فلگ‌های CLI
- `config.json.example` — نمونهٔ کانفیگ کلاینت
- `server-config.json.example` — نمونهٔ کانفیگ سرور
- `README.md` — README انگلیسی
- `README-fa.md` — README فارسی

---

## نصب پیش‌نیازها روی Ubuntu / Debian

```bash
sudo apt update
sudo apt install -y curl wget ca-certificates libpcap-dev tcpdump jq systemd libcap2-bin net-tools iproute2 dnsutils
uname -m
uname -a
```

---

## دانلود و نصب مستقیم از Releases

### نصب روی Linux AMD64

```bash
sudo apt update
sudo apt install -y curl wget ca-certificates libpcap-dev tcpdump jq systemd libcap2-bin net-tools iproute2 dnsutils
sudo install -d -m 0755 /usr/local/bin
sudo wget -O /usr/local/bin/spoof https://github.com/ParsaKSH/spoof-tunnel/releases/latest/download/spoof-linux-amd64
sudo chown root:root /usr/local/bin/spoof
sudo chmod 0755 /usr/local/bin/spoof
/usr/local/bin/spoof -version || /usr/local/bin/spoof -h
```

### نصب روی Linux ARM64

```bash
sudo apt update
sudo apt install -y curl wget ca-certificates libpcap-dev tcpdump jq systemd libcap2-bin net-tools iproute2 dnsutils
sudo install -d -m 0755 /usr/local/bin
sudo wget -O /usr/local/bin/spoof https://github.com/ParsaKSH/spoof-tunnel/releases/latest/download/spoof-linux-arm64
sudo chown root:root /usr/local/bin/spoof
sudo chmod 0755 /usr/local/bin/spoof
/usr/local/bin/spoof -version || /usr/local/bin/spoof -h
```

### نصب دستی از روی باینری دانلودشده و آپلودشده روی روت سرور

اگر فایل باینری را جداگانه دانلود کرده‌اید و روی سرور در مسیر `/root/` آپلود کرده‌اید، یکی از دو دستور نهایی زیر را اجرا کنید.

#### اگر سرور Linux AMD64 است

```bash
sudo install -d -m 0755 /usr/local/bin
sudo install -m 0755 /root/spoof-linux-amd64 /usr/local/bin/spoof
sudo chown root:root /usr/local/bin/spoof
sudo chmod 0755 /usr/local/bin/spoof
/usr/local/bin/spoof -version || /usr/local/bin/spoof -h
```

#### اگر سرور Linux ARM64 است

```bash
sudo install -d -m 0755 /usr/local/bin
sudo install -m 0755 /root/spoof-linux-arm64 /usr/local/bin/spoof
sudo chown root:root /usr/local/bin/spoof
sudo chmod 0755 /usr/local/bin/spoof
/usr/local/bin/spoof -version || /usr/local/bin/spoof -h
```

---

## ساخت کلیدها

روی هر سمت یک بار اجرا کنید:

```bash
/usr/local/bin/spoof -generate-keys
```

قاعدهٔ استفاده از کلیدها:

- `private_key` = کلید خصوصی همان ماشین
- `peer_public_key` = کلید عمومی سمت مقابل

ترتیب پیشنهادی:

1. روی سرور کلید بسازید.
2. روی کلاینت کلید بسازید.
3. فقط public key ها را بین دو طرف ردوبدل کنید.
4. private key هر سمت فقط روی همان میزبان بماند.

---

## ساخت دایرکتوری‌های لازم

```bash
sudo install -d -m 0700 /etc/spoof-tunnel
sudo install -d -m 0755 /var/log/spoof-tunnel
sudo touch /var/log/spoof-tunnel/server.log
sudo touch /var/log/spoof-tunnel/client.log
sudo chown root:root /var/log/spoof-tunnel/server.log /var/log/spoof-tunnel/client.log
sudo chmod 0644 /var/log/spoof-tunnel/server.log /var/log/spoof-tunnel/client.log
```

---

## منطق اجرای برنامه

برنامه subcommand جدا برای کلاینت و سرور ندارد. حالت اجرا از فیلد `mode` داخل فایل JSON خوانده می‌شود:

- `"mode": "server"`
- `"mode": "client"`

روش صحیح اجرا:

```bash
sudo /usr/local/bin/spoof -config /etc/spoof-tunnel/server.json
```

یا:

```bash
sudo /usr/local/bin/spoof -config /etc/spoof-tunnel/client.json
```

---

## کانفیگ کامل سرور

فایل `/etc/spoof-tunnel/server.json` را دقیقاً با این محتوا بسازید و فقط مقادیر داخل کوتیشن‌ها را با مقادیر واقعی خودتان جایگزین کنید:

```bash
sudo tee /etc/spoof-tunnel/server.json > /dev/null <<'JSON'
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
JSON
```

---

## کانفیگ کامل کلاینت

فایل `/etc/spoof-tunnel/client.json` را دقیقاً با این محتوا بسازید و فقط مقادیر داخل کوتیشن‌ها را با مقادیر واقعی خودتان جایگزین کنید:

```bash
sudo tee /etc/spoof-tunnel/client.json > /dev/null <<'JSON'
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
JSON
```

---

## اجرای دستی برای تست اولیه

### اجرای سرور

```bash
sudo /usr/local/bin/spoof -config /etc/spoof-tunnel/server.json
```

### اجرای کلاینت

```bash
sudo /usr/local/bin/spoof -config /etc/spoof-tunnel/client.json
```

---

## اجرای بدون sudo کامل با capability

```bash
sudo setcap cap_net_raw+ep /usr/local/bin/spoof
getcap /usr/local/bin/spoof
```

خروجی مورد انتظار:

```bash
/usr/local/bin/spoof cap_net_raw=ep
```

---

## ساخت سرویس systemd برای سرور

فایل سرویس را بسازید:

```bash
sudo tee /etc/systemd/system/spoof-tunnel-server.service > /dev/null <<'SERVICE'
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
SERVICE
```

سرویس را فعال و اجرا کنید:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now spoof-tunnel-server
sudo systemctl status spoof-tunnel-server --no-pager -l
```

---

## ساخت سرویس systemd برای کلاینت

فایل سرویس را بسازید:

```bash
sudo tee /etc/systemd/system/spoof-tunnel-client.service > /dev/null <<'SERVICE'
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
SERVICE
```

سرویس را فعال و اجرا کنید:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now spoof-tunnel-client
sudo systemctl status spoof-tunnel-client --no-pager -l
```

---

## دستورات مدیریت سرویس

### سرور

```bash
sudo systemctl restart spoof-tunnel-server
sudo systemctl stop spoof-tunnel-server
sudo systemctl start spoof-tunnel-server
sudo systemctl disable spoof-tunnel-server
sudo journalctl -u spoof-tunnel-server -f
```

### کلاینت

```bash
sudo systemctl restart spoof-tunnel-client
sudo systemctl stop spoof-tunnel-client
sudo systemctl start spoof-tunnel-client
sudo systemctl disable spoof-tunnel-client
sudo journalctl -u spoof-tunnel-client -f
```

---

## Health Check بعد از راه‌اندازی

### بررسی اینکه کلاینت روی پورت محلی بالا آمده است

```bash
sudo ss -lntp | grep 1080
```

### بررسی اینکه پروسه در حال اجرا است

```bash
ps -ef | grep spoof | grep -v grep
```

### بررسی اینکه systemd سرویس را active نگه داشته است

```bash
sudo systemctl is-active spoof-tunnel-server
sudo systemctl is-active spoof-tunnel-client
```

### تست درخواست از طریق SOCKS5 محلی

```bash
curl --socks5-hostname 127.0.0.1:1080 https://example.com/ -I
```

---

## استفاده از SOCKS5 محلی بعد از بالا آمدن کلاینت

بعد از بالا آمدن کلاینت، endpoint محلی پیش‌فرض این است:

```text
127.0.0.1:1080
```

### تست با curl

```bash
curl --socks5-hostname 127.0.0.1:1080 https://example.com/
```

### تنظیم متغیر محیطی برای ابزارهای CLI

```bash
export ALL_PROXY=socks5h://127.0.0.1:1080
export all_proxy=socks5h://127.0.0.1:1080
export HTTP_PROXY=socks5h://127.0.0.1:1080
export HTTPS_PROXY=socks5h://127.0.0.1:1080
export http_proxy=socks5h://127.0.0.1:1080
export https_proxy=socks5h://127.0.0.1:1080
```

### تست بعد از ست‌کردن متغیرها

```bash
curl https://example.com/
```

### استفاده با git

```bash
git config --global http.proxy socks5h://127.0.0.1:1080
git config --global https.proxy socks5h://127.0.0.1:1080
```

### حذف پراکسی git در صورت نیاز

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```

---

## لاگ‌گیری

سطوح معتبر لاگ:

- `debug`
- `info`
- `warn`
- `error`

دستورهای بررسی لاگ:

```bash
sudo tail -f /var/log/spoof-tunnel/server.log
sudo tail -f /var/log/spoof-tunnel/client.log
sudo journalctl -u spoof-tunnel-server -f
sudo journalctl -u spoof-tunnel-client -f
```

---

## عیب‌یابی

### خطای config

موارد زیر را بررسی کنید:

- `mode` معتبر باشد
- در حالت `client`، مقادیر `server.address` و `server.port` تنظیم شده باشند
- `crypto.private_key` مقدار داشته باشد
- `crypto.peer_public_key` مقدار داشته باشد
- در حالت `server`، مقدار `spoof.client_real_ip` یا `spoof.client_real_ipv6` تنظیم شده باشد
- همهٔ IP ها معتبر باشند

### هشدار مربوط به دسترسی

اگر هشدار مربوط به raw socket دریافت کردید، برنامه را با `root` اجرا کنید یا `CAP_NET_RAW` را روی باینری اعمال کنید.

### خطای کلیدها

اگر parse کلیدها خطا داد، دوباره کلید بسازید:

```bash
/usr/local/bin/spoof -generate-keys
```

### ناپایداری مربوط به MTU

اگر ارتباط ناپایدار بود، مقدار `mtu` را از `1400` به `1300` یا کمتر کاهش دهید.

### خطای FEC

اگر `fec.enabled` برابر `true` است:

- `data_shards` باید حداقل `1` باشد
- `parity_shards` باید حداقل `1` باشد
- مجموع آن‌ها نباید بیشتر از `256` شود

### کلاینت بالا است ولی پورت محلی باز نشده است

```bash
sudo systemctl status spoof-tunnel-client --no-pager -l
sudo journalctl -u spoof-tunnel-client -n 100 --no-pager
sudo ss -lntp | grep 1080
cat /etc/spoof-tunnel/client.json | jq .
```
---

## چک‌لیست سرور

- باینری در `/usr/local/bin/spoof` نصب شده باشد
- کلید خصوصی سرور ساخته شده باشد
- کلید عمومی کلاینت در `peer_public_key` قرار گرفته باشد
- فایل `/etc/spoof-tunnel/server.json` ساخته شده باشد
- مسیر لاگ وجود داشته باشد
- سرویس `spoof-tunnel-server` فعال شده باشد

---

## چک‌لیست کلاینت

- باینری در `/usr/local/bin/spoof` نصب شده باشد
- کلید خصوصی کلاینت ساخته شده باشد
- کلید عمومی سرور در `peer_public_key` قرار گرفته باشد
- فایل `/etc/spoof-tunnel/client.json` ساخته شده باشد
- مسیر لاگ وجود داشته باشد
- سرویس `spoof-tunnel-client` فعال شده باشد
- پورت `127.0.0.1:1080` در `ss` دیده شود
- تست `curl --socks5-hostname 127.0.0.1:1080 https://example.com/` موفق باشد

---

## محدودیت‌های مهم

- بسیاری از دیتاسنترها و cloud providerها خروجی با Source IP جعلی را در سطح hypervisor، ToR switch یا ACL شبکه مسدود می‌کنند.
- بعضی سرورها اجازهٔ raw socket یا capability لازم را فقط با `root` می‌دهند.
- بعضی محیط‌ها `iptables` ندارند و فقط `nftables` فعال است.
- بعضی kernel ها یا image ها محدودیت‌هایی برای packet capture و raw networking دارند.
- اگر شبکهٔ دو طرف با نیازهای این پروژه سازگار نباشد، صرفاً با تغییر README مشکل حل نمی‌شود.

---

## لایسنس

این مخزن با مجوز Apache-2.0 منتشر شده است.
