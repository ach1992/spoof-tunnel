# Spoof Tunnel

> **فقط برای محیط‌های آزمایشگاهی، پژوهشی و تست روی شبکه‌هایی که مالک آن هستید یا مجوز صریح دارید.**
>
> این فایل نسخهٔ فارسیِ بازنویسی‌شده و کامل‌تر README پروژه است و روی نصب، ساخت، پیکربندی، اجرای سرویس، لاگ‌گیری و عیب‌یابی در **Ubuntu / Debian** تمرکز دارد. عمداً از ارائهٔ راهنمای استفاده روی شبکه‌ها یا سیاست‌هایی که تحت مالکیت یا اختیار شما نیستند خودداری شده است.

[English](README.md)

## این پروژه چیست؟

Spoof Tunnel یک تونل لایه ۳ / لایه ۴ با مدل client/server است که از raw packet، رمزنگاری و مکانیزم mutual spoofing استفاده می‌کند. README اصلی مخزن معماری کلی، لایهٔ اطمینان، مالتی‌پلکسینگ و رمزنگاری را توضیح می‌دهد و کد پروژه نیز نشان می‌دهد که برنامه به زبان Go نوشته شده و در یک باینری واحد، هر دو حالت client و server را پشتیبانی می‌کند.

## این راهنما چه چیزهایی را کامل می‌کند؟

README فعلی بیشتر روی توضیح معماری متمرکز است، اما برای کاربر نهایی این بخش‌ها را کامل و عملی توضیح نمی‌دهد:

- نصب پیش‌نیازها روی Ubuntu / Debian
- بیلد صحیح باینری
- ساخت کلیدها با فلگ واقعی برنامه
- ساخت فایل کانفیگ کلاینت و سرور
- روش درست اجرای برنامه
- نحوهٔ دادن دسترسی لازم برای raw socket
- اجرای برنامه با `systemd`
- بررسی لاگ و عیب‌یابی خطاهای رایج

این فایل برای پوشش همین کمبودها نوشته شده است.

---

## نکات مهم قبل از شروع

1. **فقط روی سیستم‌ها و شبکه‌هایی استفاده کنید که مالک آن هستید یا مجوز تست دارید.**
2. **Raw socket به دسترسی بالا نیاز دارد.** خود کد هم هشدار می‌دهد که اجرای بدون root ممکن است شکست بخورد.
3. **فایل config معتبر الزامی است.** برنامه قبل از شروع، mode، transport، آدرس‌ها، فیلدهای spoof، کلیدها و چند بخش دیگر را اعتبارسنجی می‌کند.
4. **کلاینت و سرور subcommand جدا ندارند.** حالت اجرا از داخل فایل JSON و با فیلد `mode` تعیین می‌شود و برنامه با فلگ `-config` اجرا می‌شود. مثال‌های فعلی README که از `server -c ...` یا `client -c ...` استفاده کرده‌اند با `main.go` هماهنگ نیستند.
5. **ساخت کلید با `-generate-keys` انجام می‌شود.** در README فعلی این بخش هم با کد یکی نیست.

---

## سیستم‌عامل هدف

این راهنما برای این نسخه‌ها نوشته شده است:

- Ubuntu 20.04+
- Ubuntu 22.04+
- Ubuntu 24.04+
- Debian 11+
- Debian 12+

پیش‌نیاز کلی:

- لینوکس 64 بیتی
- دسترسی root یا امکان دادن capability به باینری
- نصب Go برای بیلد از سورس
- وجود `systemd` اگر بخواهید سرویس دائمی بسازید

---

## ساختار مهم مخزن

فایل‌های مهم داخل پروژه:

- `cmd/spoof/main.go` — نقطهٔ ورود برنامه و فلگ‌های CLI
- `config.json.example` — نمونهٔ کانفیگ کلاینت
- `server-config.json.example` — نمونهٔ کانفیگ سرور
- `README.md` — README انگلیسی
- `README-fa.md` — README فارسی

---

## ۱) نصب پیش‌نیازها روی Ubuntu / Debian

```bash
sudo apt update
sudo apt install -y git curl wget ca-certificates build-essential pkg-config \
  golang-go libpcap-dev tcpdump jq systemd
```

توضیح:

- README و ساختار پروژه نشان می‌دهند که در پیاده‌سازی از `gopacket` و `pcap` استفاده شده است.
- اگر Go را جداگانه و با نسخهٔ جدیدتر نصب کرده‌اید، بخش `golang-go` را می‌توانید حذف کنید.

بررسی نسخه‌ها:

```bash
go version
uname -a
```

---

## ۲) دریافت سورس پروژه

```bash
git clone https://github.com/ParsaKSH/spoof-tunnel.git
cd spoof-tunnel
```

اگر می‌خواهید نسخهٔ release مشخصی را بیلد کنید، قبل از build روی همان tag checkout کنید. صفحهٔ GitHub پروژه وجود release را نشان می‌دهد.

---

## ۳) بیلد کردن باینری

برای Linux AMD64:

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o spoof ./cmd/spoof/
```

این همان دستور build است که در README فعلی هم آمده.

نصب اختیاری در `/usr/local/bin`:

```bash
sudo install -m 0755 spoof /usr/local/bin/spoof
```

بررسی اجرای برنامه:

```bash
spoof -version
```

فلگ `-version` در `main.go` تعریف شده است.

---

## ۴) ساخت کلیدها به روش درست

روی هر سمت یک بار اجرا کنید:

```bash
./spoof -generate-keys
```

خروجی شامل این موارد است:

- **Private Key** برای همان ماشین
- **Public Key** برای اشتراک با سمت مقابل

قاعدهٔ استفاده از کلیدها:

- `private_key` = کلید خصوصی همان سمت
- `peer_public_key` = کلید عمومی سمت مقابل

این رفتار هم در `main.go` و هم در اعتبارسنجی config مشخص است.

ترتیب پیشنهادی:

1. روی سرور کلید بسازید.
2. روی کلاینت کلید بسازید.
3. فقط public key ها را بین دو طرف ردوبدل کنید.
4. private key هر سمت فقط روی همان میزبان بماند.

---

## ۵) ساخت دایرکتوری برای config و log

```bash
sudo mkdir -p /etc/spoof-tunnel
sudo mkdir -p /var/log/spoof-tunnel
sudo chmod 700 /etc/spoof-tunnel
```

نام‌گذاری پیشنهادی:

- سرور: `/etc/spoof-tunnel/server.json`
- کلاینت: `/etc/spoof-tunnel/client.json`

---

## ۶) منطق انتخاب حالت اجرا

برنامه subcommand جدا برای کلاینت و سرور ندارد. حالت از فیلد `mode` داخل JSON خوانده می‌شود:

- `"mode": "server"`
- `"mode": "client"`

پس روش درست اجرا این است:

```bash
sudo ./spoof -config /path/to/config.json
```

یا اگر باینری را global نصب کرده‌اید:

```bash
sudo spoof -config /path/to/config.json
```

---

## ۷) نمونهٔ کامل کانفیگ کلاینت

فایل `config.json.example` در مخزن وجود دارد، اما نسخهٔ فشرده و کم‌توضیحی است. نسخهٔ زیر برای شروع خواناتر است و JSON معتبر هم باقی می‌ماند:

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

چرا این فایل با `config.json.example` کمی فرق دارد:

- فایل نمونهٔ مخزن خیلی فشرده است و بعضی فیلدها را صریح نشان نمی‌دهد
- فیلد `send_rate_limit` در `PerformanceConfig` در کد وجود دارد، ولی در README فعلی عملاً مستند نشده است.

### معنی فیلدهای مهم در کلاینت

- `listen.address` / `listen.port`: آدرس و پورتی که پروکسی محلی SOCKS5 روی آن باز می‌شود. اگر مقدار ندهید در کد برای حالت کلاینت `127.0.0.1:1080` پیش‌فرض می‌شود.
- `server.address` / `server.port`: آدرس واقعی سمت سرور
- `spoof.source_ip`: شناسهٔ spoof سمت کلاینت
- `spoof.peer_spoof_ip`: آی‌پی spoof مورد انتظار از سمت سرور
- `crypto.private_key`: کلید خصوصی خود کلاینت
- `crypto.peer_public_key`: کلید عمومی سرور

---

## ۸) نمونهٔ کامل کانفیگ سرور

در مخزن فایل `server-config.json.example` هم وجود دارد. نسخهٔ تمیزتر و مناسب شروع:

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

### نکتهٔ مهم مخصوص سرور

در حالت server، اگر `client_real_ip` یا `client_real_ipv6` را نگذارید، اعتبارسنجی config خطا می‌دهد و برنامه بالا نمی‌آید.

---

## ۹) مرجع کامل فیلدهای کانفیگ

### فیلدهای اجباری

#### اجباری روی هر دو سمت

- `mode`
- `transport.type`
- حداقل یکی از این‌ها: `spoof.source_ip` یا `spoof.source_ipv6`
- `crypto.private_key`
- `crypto.peer_public_key`

#### اجباری در حالت client

- `server.address`
- `server.port`

#### اجباری در حالت server

- `spoof.client_real_ip` یا `spoof.client_real_ipv6`

### بخش transport

- `transport.type`: در validation کد مقادیر `udp`، `icmp` و `raw` پذیرفته می‌شوند. README فعلی بیشتر فقط `udp` و `icmp` را پوشش داده است.
- `transport.icmp_mode`: برای ICMP می‌تواند `echo` یا `reply` باشد
- `transport.protocol_number`: فقط وقتی `type=raw` باشد استفاده می‌شود و باید بین 1 تا 255 باشد.

### بخش listen

- `listen.address`: آی‌پی bind محلی
- `listen.port`: پورت bind
- در کد برای بسیاری از فیلدها مقدار پیش‌فرض تنظیم می‌شود؛ برای کلاینت معمولاً `127.0.0.1:1080` نقطهٔ شروع است.

### بخش performance

- `buffer_size`: اندازهٔ بافر اصلی بسته‌ها
- `mtu`: اندازهٔ payload قبل از encapsulation؛ اگر fragmentation دیدید مقدار را کمتر کنید
- `session_timeout`: timeout کلی نشست
- `workers`: تعداد worker / goroutine پردازش
- `read_buffer` / `write_buffer`: اندازهٔ بافرهای socket
- `send_rate_limit`: محدودیت نرخ ارسال بر حسب packet در ثانیه؛ این فیلد در کد وجود دارد.

### بخش reliability

اگر بعضی فیلدهای reliability را نگذارید، کد برای آن‌ها مقدار پیش‌فرض می‌گذارد.

### بخش FEC

اگر `fec.enabled=true` باشد:

- `data_shards >= 1`
- `parity_shards >= 1`
- مجموع آن‌ها نباید بیشتر از 256 شود.

### بخش keepalive

اگر بعضی مقادیر را نگذارید، کد برای interval و timeout مقدار پیش‌فرض تعیین می‌کند.

---

## ۱۰) اول به صورت دستی اجرا کنید

### سرور

```bash
sudo spoof -config /etc/spoof-tunnel/server.json
```

### کلاینت

```bash
sudo spoof -config /etc/spoof-tunnel/client.json
```

انتظار از رفتار برنامه:

- سرور باید mode، transport و spoof source IP را در لاگ نشان بدهد
- کلاینت باید SOCKS5 proxy، سرور مقصد و spoof source IP را در لاگ نشان بدهد
- در صورت شروع موفق، کلاینت روی آدرس محلی تعیین‌شده یک SOCKS5 proxy باز می‌کند؛ معمولاً `127.0.0.1:1080`

---

## ۱۱) اجرا بدون sudo کامل با استفاده از capability

اگر نخواهید هر بار با `sudo` اجرا کنید:

```bash
sudo setcap cap_net_raw+ep /usr/local/bin/spoof
getcap /usr/local/bin/spoof
```

نمونهٔ خروجی:

```bash
/usr/local/bin/spoof cap_net_raw=ep
```

اگر در محیط شما همچنان privilege بیشتری برای packet handling لازم باشد، اجرای مستقیم با root ساده‌تر است.

---

## ۱۲) فایل‌های نمونهٔ `systemd`

### سرویس سرور

فایل `/etc/systemd/system/spoof-tunnel-server.service`:

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

### سرویس کلاینت

فایل `/etc/systemd/system/spoof-tunnel-client.service`:

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

فعال‌سازی و اجرا:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now spoof-tunnel-server
# یا
sudo systemctl enable --now spoof-tunnel-client
```

بررسی وضعیت:

```bash
systemctl status spoof-tunnel-server
journalctl -u spoof-tunnel-server -f
```

---

## ۱۳) استفاده از SOCKS5 محلی روی کلاینت

بعد از بالا آمدن کلاینت، برنامه‌ها می‌توانند از SOCKS5 محلی استفاده کنند.

نمونه با `curl`:

```bash
curl --socks5-hostname 127.0.0.1:1080 https://example.com/
```

اگر `listen.address` یا `listen.port` را تغییر داده‌اید، همان endpoint جدید را استفاده کنید.

---

## ۱۴) لاگ‌گیری

سطوح لاگ معتبر:

- `debug`
- `info`
- `warn`
- `error`

نمونهٔ ثبت لاگ در فایل:

```json
"logging": {
  "level": "debug",
  "file": "/var/log/spoof-tunnel/client.log"
}
```

اگر `file` خالی باشد، خروجی لاگ روی stdout/stderr می‌ماند.

دستورهای مفید:

```bash
tail -f /var/log/spoof-tunnel/client.log
tail -f /var/log/spoof-tunnel/server.log
journalctl -u spoof-tunnel-client -f
```

---

## ۱۵) عیب‌یابی

### برنامه بلافاصله با خطای config خارج می‌شود

اعتبارسنجی config سخت‌گیرانه است. خطاهای رایج:

- `mode` نامعتبر
- نداشتن `server.address` در حالت client
- نداشتن `crypto.private_key`
- نداشتن `crypto.peer_public_key`
- نداشتن `spoof.client_real_ip` در حالت server
- نادرست بودن syntax آی‌پی‌ها در فیلدهای listen یا spoof

### هشدار «Running without root privileges. Raw sockets may fail.»

این هشدار مستقیماً از کد می‌آید. باید با root اجرا کنید یا `CAP_NET_RAW` بدهید.

### کلاینت بالا می‌آید ولی پروکسی محلی در دسترس نیست

این موارد را چک کنید:

- `mode` واقعاً `client` باشد
- `listen.address` و `listen.port` درست باشند
- process بعد از startup فوراً exit نکرده باشد

### خطای مربوط به کلیدها

اگر parse کلیدها خطا داد، دوباره اجرا کنید:

```bash
./spoof -generate-keys
```

و مطمئن شوید هر سمت:

- private key خودش را دارد
- public key سمت مقابل را در `peer_public_key` گذاشته است

### ناپایداری مرتبط با MTU

اگر انتقال‌ها unstable هستند، `performance.mtu` را از `1400` به `1300` یا کمتر کاهش دهید و دوباره تست کنید. در README فعلی هم به اهمیت تنظیم MTU اشاره شده است.

### خطای FEC

اگر FEC فعال است، تعداد shardها باید مثبت باشد و مجموع آن‌ها از 256 بیشتر نشود.

---

## ۱۶) چک‌لیست عملیاتی

### چک‌لیست سرور

- [ ] Go و پیش‌نیازها نصب شده‌اند
- [ ] باینری build یا install شده است
- [ ] private key سرور ساخته شده است
- [ ] public key کلاینت در `peer_public_key` قرار گرفته است
- [ ] `mode=server` تنظیم شده است
- [ ] `spoof.client_real_ip` پر شده است
- [ ] مسیر log قابل نوشتن است
- [ ] برنامه با root یا capability مناسب اجرا می‌شود

### چک‌لیست کلاینت

- [ ] باینری build یا install شده است
- [ ] private key کلاینت ساخته شده است
- [ ] public key سرور در `peer_public_key` قرار گرفته است
- [ ] `mode=client` تنظیم شده است
- [ ] `server.address` و `server.port` تنظیم شده‌اند
- [ ] SOCKS5 محلی تنظیم شده است
- [ ] برنامه با root یا capability مناسب اجرا می‌شود

---

## ۱۷) ایرادهای مستندات فعلی که در این نسخه اصلاح شدند

این نسخهٔ بازنویسی‌شده این موارد را اصلاح یا روشن می‌کند:

1. **CLI صحیح برای config**: باید از `-config` استفاده شود، نه `-c`. در کد آمده `flag.String("config", ...)`.
2. **CLI صحیح برای ساخت کلید**: باید `-generate-keys` استفاده شود، نه subcommand به نام `generate-keys`.
3. **حالت اجرا از فایل config می‌آید**: subcommand جدا برای `server` یا `client` در کد پیاده‌سازی نشده است.
4. **در حالت server، فیلد `client_real_ip` الزامی است** و باید واضح مستند شود.
5. **transport نوع `raw` در validation وجود دارد** هرچند در README فعلی تقریباً پوشش داده نشده است.
6. **فیلد `send_rate_limit` در کد وجود دارد** و برای کامل شدن مستندات بهتر است ذکر شود.

---

## ۱۸) لایسنس

طبق صفحهٔ GitHub، این مخزن با مجوز Apache-2.0 منتشر شده است.

---

## ۱۹) نام پیشنهادی فایل‌ها برای آپلود روی GitHub

اگر می‌خواهید READMEهای فعلی را در fork خودتان جایگزین کنید:

- `README.md` → نسخهٔ انگلیسی
- `README-fa.md` → نسخهٔ فارسی

اگر می‌خواهید اول جداگانه بررسی کنید و بعد جایگزین کنید:

- `README.en.complete.md`
- `README-fa.complete.md`


## محدودیت‌های مهم و واقع‌بینانه

این پروژه را نمی‌توان صادقانه «قابل اجرا روی هر سرور» نامید. حتی با README کامل هم این محدودیت‌ها باقی می‌مانند:

- بسیاری از دیتاسنترها و cloud providerها خروجی با Source IP جعلی را در سطح hypervisor، ToR switch یا ACL شبکه مسدود می‌کنند.
- بعضی سرورها اجازهٔ raw socket یا قابلیت‌های لازم را فقط با root می‌دهند.
- بعضی محیط‌ها `iptables` ندارند و فقط `nftables` فعال است.
- بعضی kernel / imageها ماژول‌ها یا تنظیمات لازم برای packet capture و raw networking را ندارند.
- اگر شبکهٔ دو طرف واقعاً spoof-capable نباشد، هیچ READMEیی به تنهایی مشکل را حل نمی‌کند.

پس این README برای **Ubuntu/Debian روی سرورهایی که از نظر شبکه و دسترسی با نیازهای پروژه سازگارند** کامل و عملیاتی است، اما برای «هر سرور» تضمین مطلق نمی‌دهد.

## برای اینکه README واقعاً upload-ready باشد

در نسخه‌ای که برای GitHub آپلود می‌کنید، این موارد باید حتماً رعایت شود:

- هیچ citation داخلی یا نشانهٔ ابزار در متن نباشد.
- Quick Start خیلی کوتاه و copy/paste-ready در ابتدای فایل باشد.
- بخش «Supported / Unsupported Environments» واضح باشد.
- بخش «Before You Open an Issue» برای خطاهای رایج اضافه شود.
- برای `iptables` و `nftables` اگر لازم است هر دو مسیر توضیح داده شوند.
- نمونهٔ systemd unit کاملاً تست‌شده و یکدست باشد.
- یک بخش FAQ کوتاه برای خطاهای متداول اضافه شود.
