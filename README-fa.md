# Spoof Tunnel

> **فقط برای محیط‌های آزمایشگاهی، پژوهشی و تست روی سرورها و شبکه‌هایی که مالک آن هستید یا مجوز صریح دارید.**
>
> این فایل نسخهٔ نهایی فارسی README برای **Ubuntu / Debian** است و بر پایهٔ **کد واقعی برنامه** تنظیم شده است، نه صرفاً مثال‌های README انگلیسی. در چند موردی که READMEهای موجود مخزن با سورس فعلی ناهماهنگ هستند (مثل ساخت کلید و بعضی فلگ‌ها)، این فایل بر اساس وضعیت فعلی سورس اصلاح شده است.

[English](README.md)

---

## این پروژه چیست؟

اینجا Spoof Tunnel یک تونل لایه ۳ / لایه ۴ با مدل client/server است که از raw packet، رمزنگاری و mutual spoofing استفاده می‌کند. برنامه در یک باینری واحد اجرا می‌شود و حالت client یا server از داخل فایل کانفیگ مشخص می‌شود.

---

## نکات مهم قبل از شروع

1. هر دو سرور باید امکان ارسال بسته با source spoofed را داشته باشند.
2. اجرای برنامه به دسترسی root نیاز دارد.
3. باینری یک فایل واحد است و subcommand جدا برای `server` و `client` ندارد؛ حالت اجرا از داخل `mode` در فایل کانفیگ مشخص می‌شود.
4. طبق کد فعلی برنامه، فلگ درست اجرای کانفیگ **`--config`** یا معادل کوتاه آن **`-c`** است.
5. طبق سورس فعلی، دستور داخلیِ قابل‌استفاده برای ساخت کلید در CLI ثبت نشده است؛ بنابراین در بخش «ساخت کلیدها» پایین‌تر، روش عملی و قابل اجرا برای تولید کلیدهای سازگار آورده شده است.
6. نام فایل‌های release برای لینوکس به شکل زیر است:
   - `spoof-linux-amd64`
   - `spoof-linux-arm64`

---

## سیستم‌عامل هدف

- Ubuntu 20.04+
- Ubuntu 22.04+
- Ubuntu 24.04+
- Debian 11+
- Debian 12+

---

## تشخیص معماری سرور

```bash
uname -m
```

اگر خروجی یکی از موارد زیر بود، فایل مناسب را بگیرید:

- `x86_64` → `spoof-linux-amd64`
- `aarch64` → `spoof-linux-arm64`

---

## نصب پیش‌نیازها

```bash
sudo apt update
sudo apt install -y curl wget ca-certificates libpcap-dev tcpdump jq systemd libcap2-bin net-tools iproute2 dnsutils
sudo install -d -m 0755 /usr/local/bin
sudo mkdir -p /etc/spoof-tunnel
sudo mkdir -p /var/log/spoof-tunnel
sudo chmod 700 /etc/spoof-tunnel
```

---

## نصب از Releases بدون نیاز به Build

### نصب روی Linux AMD64

```bash
sudo wget -O /usr/local/bin/spoof https://github.com/ParsaKSH/spoof-tunnel/releases/latest/download/spoof-linux-amd64
sudo chown root:root /usr/local/bin/spoof
sudo chmod 0755 /usr/local/bin/spoof
/usr/local/bin/spoof --version
```

### نصب روی Linux ARM64

```bash
sudo wget -O /usr/local/bin/spoof https://github.com/ParsaKSH/spoof-tunnel/releases/latest/download/spoof-linux-arm64
sudo chown root:root /usr/local/bin/spoof
sudo chmod 0755 /usr/local/bin/spoof
/usr/local/bin/spoof --version
```

---

## نصب دستی وقتی باینری را دانلود کرده‌اید و روی `/root/` آپلود شده است

### اگر سرور AMD64 است

```bash
sudo install -m 0755 /root/spoof-linux-amd64 /usr/local/bin/spoof
sudo chown root:root /usr/local/bin/spoof
/usr/local/bin/spoof --version
```

### اگر سرور ARM64 است

```bash
sudo install -m 0755 /root/spoof-linux-arm64 /usr/local/bin/spoof
sudo chown root:root /usr/local/bin/spoof
/usr/local/bin/spoof --version
```

اگر فایل را با نام دیگری آپلود کرده‌اید، اول نام آن را ببینید:

```bash
ls -lh /root/
```

---

## ساخت کلیدها

روی هر سمت یک بار اجرا کنید:

```bash
/usr/local/bin/spoof -generate-keys
```

قاعدهٔ استفاده از کلیدها:

- `private_key` = کلید خصوصی همان سمت
- `peer_public_key` = کلید عمومی سمت مقابل

ترتیب کار:

1. روی سرور کلید بسازید.
2. روی کلاینت کلید بسازید.
3. فقط public keyها را بین دو سمت جابه‌جا کنید.
4. private key هر سمت فقط روی همان ماشین باقی بماند.

---

## انتخاب IP مناسب برای spoof

برای `source_ip` و `peer_spoof_ip` از IP دلخواه یا IP متعلق به دیگران استفاده نکنید.

مقدار مناسب معمولاً یکی از این‌هاست:

- IP واقعی اینترفیس همان ماشین
- IP خصوصی واقعی داخل همان شبکه اگر دو سمت از همان مسیر به هم می‌رسند
- IP عمومی واقعی خود سرور، فقط وقتی همان آدرسی است که ترافیک باید با آن دیده و برگشت داده شود

برای پیدا کردن IP مناسب:

```bash
ip -4 addr show
ip route
```

قاعدهٔ عملی:

- در کلاینت، `source_ip` را از IP واقعی سمت کلاینت بگذارید
- در سرور، `source_ip` را از IP واقعی سمت سرور بگذارید
- `peer_spoof_ip` هر سمت باید با `source_ip` سمت مقابل هماهنگ باشد

از IPهای تصادفی مثل `6.6.6.6` استفاده نکنید.

---

## نمونهٔ کامل کانفیگ کلاینت

فایل زیر را دقیقاً در مسیر `/etc/spoof-tunnel/client.json` ایجاد کنید و فقط مقادیر مشخص‌شده را با مقادیر واقعی خودتان جایگزین کنید:

```bash
sudo mkdir -p /etc/spoof-tunnel && sudo tee /etc/spoof-tunnel/client.json> /dev/null <<'EOF'
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
    "mtu": 1180,
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
EOF
```

---

## نمونهٔ کامل کانفیگ سرور

فایل زیر را دقیقاً در مسیر `/etc/spoof-tunnel/server.json` ایجاد کنید و فقط مقادیر مشخص‌شده را با مقادیر واقعی خودتان جایگزین کنید:

```bash
sudo mkdir -p /etc/spoof-tunnel && sudo tee /etc/spoof-tunnel/server.json > /dev/null <<'EOF'
{
  "mode": "server",
  "transport": {
    "type": "udp",
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
    "buffer_size": 65535,
    "mtu": 1180,
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
    "file": "/var/log/spoof-tunnel/server.log"
  }
}
EOF
```

---

## اعتبارسنجی فایل کانفیگ

```bash
jq . /etc/spoof-tunnel/client.json >/dev/null && echo OK
jq . /etc/spoof-tunnel/server.json >/dev/null && echo OK
```

---

## اجرای دستی روی سرور

```bash
sudo /usr/local/bin/spoof --config /etc/spoof-tunnel/server.json
```

## اجرای دستی روی کلاینت

```bash
sudo /usr/local/bin/spoof --config /etc/spoof-tunnel/client.json
```

پس از بالا آمدن کلاینت، پروکسی محلی SOCKS5 به‌صورت پیش‌فرض روی آدرس زیر در دسترس خواهد بود:

```text
127.0.0.1:1080
```

---

## ساخت سرویس systemd برای سرور

```bash
sudo tee /etc/systemd/system/spoof-server.service >/dev/null <<'UNIT'
[Unit]
Description=Spoof Tunnel Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/local/bin/spoof --config /etc/spoof-tunnel/server.json
Restart=always
RestartSec=3
LimitNOFILE=1048576
AmbientCapabilities=CAP_NET_RAW CAP_NET_ADMIN
CapabilityBoundingSet=CAP_NET_RAW CAP_NET_ADMIN
NoNewPrivileges=false

[Install]
WantedBy=multi-user.target
UNIT
```

فعال‌سازی سرویس:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now spoof-server.service
sudo systemctl status spoof-server.service --no-pager -l
```

---

## ساخت سرویس systemd برای کلاینت

```bash
sudo tee /etc/systemd/system/spoof-client.service >/dev/null <<'UNIT'
[Unit]
Description=Spoof Tunnel Client
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/local/bin/spoof --config /etc/spoof-tunnel/client.json
Restart=always
RestartSec=3
LimitNOFILE=1048576
AmbientCapabilities=CAP_NET_RAW CAP_NET_ADMIN
CapabilityBoundingSet=CAP_NET_RAW CAP_NET_ADMIN
NoNewPrivileges=false

[Install]
WantedBy=multi-user.target
UNIT
```

فعال‌سازی سرویس:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now spoof-client.service
sudo systemctl status spoof-client.service --no-pager -l
```

---

## فرمان‌های مدیریت سرویس

### سرور

```bash
sudo systemctl restart spoof-server.service
sudo systemctl stop spoof-server.service
sudo systemctl start spoof-server.service
sudo journalctl -u spoof-server.service -f
```

### کلاینت

```bash
sudo systemctl restart spoof-client.service
sudo systemctl stop spoof-client.service
sudo systemctl start spoof-client.service
sudo journalctl -u spoof-client.service -f
```

---

## بررسی پورت و فرآیند

```bash
sudo ss -tulpn | grep spoof
ps aux | grep spoof | grep -v grep
```

---

## تست پروکسی محلی روی کلاینت

بعد از بالا آمدن کلاینت، تست کنید که SOCKS5 محلی بالا آمده باشد:

```bash
curl --socks5 127.0.0.1:1080 https://ifconfig.me
```

یا:

```bash
curl --proxy socks5h://127.0.0.1:1080 https://example.com -I
```

---

## لاگ‌ها

### فایل لاگ مستقیم

```bash
sudo tail -f /var/log/spoof-tunnel/server.log
sudo tail -f /var/log/spoof-tunnel/client.log
```

### journalctl

```bash
sudo journalctl -u spoof-server.service -n 100 --no-pager
sudo journalctl -u spoof-client.service -n 100 --no-pager
```

---

## عیب‌یابی سریع

### باینری اجرا نمی‌شود

```bash
ls -l /usr/local/bin/spoof
file /usr/local/bin/spoof
uname -m
```

اگر معماری سیستم `x86_64` است باید نسخهٔ `spoof-linux-amd64` نصب شده باشد.
اگر معماری سیستم `aarch64` است باید نسخهٔ `spoof-linux-arm64` نصب شده باشد.

### خطای `No such file or directory`

معمولاً یکی از این موارد است:

- فایل در مسیر `/usr/local/bin/spoof` وجود ندارد
- باینری معماری اشتباه دارد
- فایل executable نشده است

### خطای مربوط به permission

برنامه را با root اجرا کنید:

```bash
sudo /usr/local/bin/spoof --config /etc/spoof-tunnel/server.json
sudo /usr/local/bin/spoof --config /etc/spoof-tunnel/client.json
```

### کانفیگ اشتباه است

```bash
jq . /etc/spoof-tunnel/server.json
jq . /etc/spoof-tunnel/client.json
```

---

## حذف کامل

### حذف سرویس‌ها

```bash
sudo systemctl disable --now spoof-server.service || true
sudo systemctl disable --now spoof-client.service || true
sudo rm -f /etc/systemd/system/spoof-server.service
sudo rm -f /etc/systemd/system/spoof-client.service
sudo systemctl daemon-reload
```

### حذف فایل‌ها

```bash
sudo rm -f /usr/local/bin/spoof
sudo rm -rf /etc/spoof-tunnel
sudo rm -rf /var/log/spoof-tunnel
```

---

## جمع‌بندی

فرمان‌های اصلی این پروژه بر اساس سورس فعلی برنامه این‌ها هستند:

### بررسی نسخه

```bash
/usr/local/bin/spoof --version
```

### ساخت کلید

```bash
go run /tmp/spoof-keygen.go
```

> اگر خواستید بعداً این بخش را داخل خود باینری هم داشته باشید، باید یک ساب‌کامند مثل `keygen` به `cmd/spoof/main.go` اضافه شود؛ در snapshot فعلیِ سورس این ساب‌کامند وجود ندارد.

### اجرای سرور

```bash
sudo /usr/local/bin/spoof --config /etc/spoof-tunnel/server.json
```

### اجرای کلاینت

```bash
sudo /usr/local/bin/spoof --config /etc/spoof-tunnel/client.json
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

اگر parse کلیدها خطا داد، دوباره کلید بسازید و دقت کنید مقدار Base64 را بدون فاصله و بدون کوتیشن اضافه داخل JSON قرار دهید:

```bash
go run /tmp/spoof-keygen.go
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
sudo systemctl status spoof-client.service --no-pager -l
sudo journalctl -u spoof-client.service -n 100 --no-pager
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
- سرویس `spoof-server.service` فعال شده باشد

---

## چک‌لیست کلاینت

- باینری در `/usr/local/bin/spoof` نصب شده باشد
- کلید خصوصی کلاینت ساخته شده باشد
- کلید عمومی سرور در `peer_public_key` قرار گرفته باشد
- فایل `/etc/spoof-tunnel/client.json` ساخته شده باشد
- مسیر لاگ وجود داشته باشد
- سرویس `spoof-client.service` فعال شده باشد
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
