# Spoof Tunnel

[English](./README.md)

**اسپوف تانل (Spoof Tunnel)** یک UDP Forwarder در لایه ۳ شبکه است که با تکنیک **جعل آی‌پی دوطرفه (Mutual IP Spoofing)** ترافیک VPN را از روی فایروال‌هایی که بر اساس Source/Destination IP فیلتر می‌کنند عبور می‌دهد. هر دو سمت ارتباط، فیلد `Source IP` پکت‌ها را جعل می‌کنند تا فایروال‌های مبتنی بر conntrack نتوانند جریان رفت و برگشت را به هم پیوند بزنند.

این پروژه در جریان **قطعی کامل اینترنت ایران در پی قیام خونین ۱۸ و ۱۹ دی ۱۴۰۴ (۸ و ۹ ژانویه ۲۰۲۶)** ساخته و تست شد، زمانی که فقط IP‌های لیست سفید می‌توانستند از فایروال لایه ۳ عبور کنند.

> ⚠️ **اسپوف تانل خودش یک VPN نیست.** ترافیک را **رمزنگاری نمی‌کند** و فقط پکت‌های UDP را با مبدا جعلی فوروارد می‌کند. حتماً باید روی آن یک VPN واقعی (WireGuard توصیه می‌شود) راه بیندازید.

---

## فهرست

1. [نحوه عملکرد](#۱-نحوه-عملکرد)
2. [پیش‌نیازها](#۲-پیش‌نیازها)
3. [بررسی قابلیت اسپوف (الزامی)](#۳-بررسی-قابلیت-اسپوف-الزامی)
4. [نصب](#۴-نصب)
5. [انتخاب IPهای جعلی](#۵-انتخاب-ipهای-جعلی)
6. [راه‌اندازی سرور خارج](#۶-راه‌اندازی-سرور-خارج)
7. [راه‌اندازی سرور ایران](#۷-راه‌اندازی-سرور-ایران)
8. [اتصال VPN Client](#۸-اتصال-vpn-client)
9. [اجرا به‌صورت سرویس systemd](#۹-اجرا-به‌صورت-سرویس-systemd)
10. [حالت‌های Transport و تنظیم](#۱۰-حالت‌های-transport-و-تنظیم)
11. [چرخش IP اسپوف](#۱۱-چرخش-ip-اسپوف)
12. [فایل کانفیگ (جایگزین CLI)](#۱۲-فایل-کانفیگ-جایگزین-cli)
13. [مرجع کامل CLI](#۱۳-مرجع-کامل-cli)
14. [عیب‌یابی](#۱۴-عیب‌یابی)
15. [نکات امنیتی](#۱۵-نکات-امنیتی)
16. [لایسنس](#۱۶-لایسنس)

---

## ۱. نحوه عملکرد

### ۱.۱ نقش‌ها

| نقش | محل اجرا | ساب‌کامند |
|------|-----------|----------|
| **`local`** | سرور ایران (نقطه ورود) | `spoof local` |
| **`remote`** | سرور خارج (نقطه خروج) | `spoof remote` |

### ۱.۲ مسیر پکت

```
            سرور ایران                            سرور خارج

  برنامه/کلاینت VPN
       │ UDP
       ▼
  127.0.0.1:5000 ──spoof local──►  send-transport (پیش‌فرض TCP)
                                   src = Client_Spoof_IP:443
                                   dst = Foreign_Real_IP:8090
                                         │
                                         ▼
                                 ──spoof remote── 0.0.0.0:8090
                                         │
                                         │ UDP forward
                                         ▼
                                 127.0.0.1:51820 (WireGuard)

                                 پاسخ WireGuard (UDP)
                                         ▼
                                 ──spoof remote── (UDP transport)
                                   src = Server_Spoof_IP:8090
                                   dst = Iran_Real_IP:5001
                                         │
  127.0.0.1:5000 ◄──spoof local──        │
       ▲ UDP
       │
  برنامه/کلاینت VPN
```

### ۱.۳ چرا اسپوف دوطرفه؟

اسپوف یک‌طرفه بی‌فایده است: اگر فقط یک سمت آی‌پی را جعل کند، سمت مقابل پاسخ را به آی‌پی جعلی می‌فرستد و چیزی به مبدا واقعی بازنمی‌گردد. اسپوف تانل از قبل آی‌پی واقعی دو طرف را به هم می‌دهد، تا هر سمت بتواند پکت‌های با مبدا جعلی را به آدرس **واقعی** سمت مقابل بفرستد. نتیجه: دو جریان یک‌طرفه در دو جهت مخالف، که conntrack فایروال‌های میانی نمی‌تواند آن‌ها را به هم گره بزند.

### ۱.۴ Transport نامتقارن

Send و Recv transport می‌توانند متفاوت باشند. مقدار پیش‌فرض `local`:‌ `send=tcp, recv=udp` و پیش‌فرض `remote`: `send=udp, recv=tcp`. یعنی:
- ایران ← خارج: شبیه ترافیک TCP خروجی از یک مبدا لیست‌سفید.
- خارج ← ایران: شبیه ترافیک UDP ورودی از یک مبدا لیست‌سفید.

ICMP و ICMPv6 هم به‌عنوان Transport پشتیبانی می‌شوند.

---

## ۲. پیش‌نیازها

- دو سرور لینوکسی (Ubuntu 20.04+، Debian 11+، RHEL/AlmaLinux 8+).
- معماری: `amd64`، `arm64`، `arm`، یا `386`.
- دسترسی root (یا `CAP_NET_RAW + CAP_NET_ADMIN`).
- **هر دو سرور باید قابلیت ارسال پکت اسپوف داشته باشند** — بسیاری از پروایدرهای ارزان‌قیمت روی لبه فیلتر می‌کنند.
- یک سرور VPN واقعی روی سرور خارج. این راهنما از **WireGuard** استفاده می‌کند، اما هر VPN مبتنی بر UDP کار می‌کند (OpenVPN UDP، Hysteria و...).
- پکیج‌های سیستمی:
  ```bash
  # Debian / Ubuntu
  sudo apt update && sudo apt install -y libpcap0.8 iptables tcpdump

  # RHEL / AlmaLinux
  sudo dnf install -y libpcap iptables tcpdump
  ```

---

## ۳. بررسی قابلیت اسپوف (الزامی)

این تست را روی **هر دو سرور** قبل از هر کار دیگری انجام دهید.

**روی سروری که می‌خواهید تست کنید:**
```bash
sudo iptables -t nat -A POSTROUTING -d <TARGET_IP> -j SNAT --to-source <FAKE_IP>
ping <TARGET_IP>
```

**روی سرور گیرنده:**
```bash
sudo tcpdump -ni any icmp
```

اگر در خروجی `tcpdump` پکت‌های ICMP با `src = <FAKE_IP>` دیدید، فرستنده قابلیت اسپوف دارد.

**پاکسازی:**
```bash
sudo iptables -t nat -D POSTROUTING -d <TARGET_IP> -j SNAT --to-source <FAKE_IP>
```

اگر اسپوف کار نکرد، پروایدر را عوض کنید (مثلاً برخی رنج‌های Hetzner، سرورهای اختصاصی OVH، یا پروایدرهایی با BCP38 ضعیف).

---

## ۴. نصب

روی **هر دو سرور** اجرا کنید.

### ۴.۱ باینری آماده (پیشنهاد می‌شود)

```bash
ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/;s/armv7l/arm/')

# v1.0.2 را با آخرین تگ از صفحه Releases جایگزین کنید
curl -L -o spoof "https://github.com/ParsaKSH/spoof-tunnel/releases/download/v1.0.2/spoof-linux-${ARCH}"

chmod +x spoof
sudo mv spoof /usr/local/bin/spoof

spoof --version
```

### ۴.۲ بیلد از سورس

```bash
sudo apt install -y golang libpcap-dev git
git clone https://github.com/ParsaKSH/spoof-tunnel.git
cd spoof-tunnel
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o spoof ./cmd/spoof/
sudo mv spoof /usr/local/bin/spoof
```

---

## ۵. انتخاب IPهای جعلی

این مقادیر را قبل از ادامه یادداشت کنید — تمام دستورات روی هر دو سرور به آن‌ها ارجاع می‌دهند.

| متغیر | معنی | مثال |
|-------|------|------|
| `IRAN_REAL_IP` | IP پابلیک واقعی سرور ایران | `5.160.x.x` |
| `FOREIGN_REAL_IP` | IP پابلیک واقعی سرور خارج | `49.13.x.x` |
| `Client_Spoof_IP` | IP جعلی که سرور ایران روی پکت‌های خروجی می‌گذارد | `203.0.113.50` |
| `Server_Spoof_IP` | IP جعلی که سرور خارج روی پکت‌های خروجی می‌گذارد | `198.51.100.10` |

IP‌های جعلی را از رنج‌هایی انتخاب کنید که احتمالاً در لیست سفید فایروال هستند (DNS Resolver های معروف، رنج‌های CDN، رنج‌های دولتی و...). این IP‌ها **نباید** روی هیچ‌جای دیگر سرورهای خودتان موجود باشند.

---

## ۶. راه‌اندازی سرور خارج

### ۶.۱ نصب WireGuard Server

```bash
sudo apt install -y wireguard
wg genkey | tee /etc/wireguard/server.key | wg pubkey > /etc/wireguard/server.pub
wg genkey | tee /etc/wireguard/client.key | wg pubkey > /etc/wireguard/client.pub
chmod 600 /etc/wireguard/*.key
```

ساخت `/etc/wireguard/wg0.conf`:
```ini
[Interface]
Address = 10.66.66.1/24
ListenPort = 51820
PrivateKey = <محتوای_/etc/wireguard/server.key>
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT

[Peer]
PublicKey = <محتوای_/etc/wireguard/client.pub>
AllowedIPs = 10.66.66.2/32
```

فعال‌سازی IP Forwarding و راه‌اندازی WireGuard:
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
sudo systemctl enable --now wg-quick@wg0
```

WireGuard اکنون روی UDP `0.0.0.0:51820` گوش می‌دهد. اسپوف تانل از همان هاست از طریق `127.0.0.1:51820` به آن دسترسی دارد.

### ۶.۲ اجرای `spoof remote`

```bash
sudo spoof remote \
  --listen-port 8090 \
  --forward 127.0.0.1:51820 \
  --client-ip <IRAN_REAL_IP> \
  --client-port 5001 \
  --spoof-ip <Server_Spoof_IP> \
  --peer-spoof-ip <Client_Spoof_IP> \
  --send-transport udp \
  --recv-transport tcp \
  --spoof-port 8090
```

نقش هر flag:
- `--listen-port 8090` — پورت دریافت پکت‌های تونل از ایران (با `--remote-port` در `local` یکی است).
- `--forward 127.0.0.1:51820` — مقصد UDP که پکت‌های decapsulate شده به آن تحویل می‌شوند (سرور WireGuard).
- `--client-ip` / `--client-port` — IP و پورت واقعی سرور ایران (پاسخ‌ها به اینجا فرستاده می‌شوند).
- `--spoof-ip` — IP جعلی که این سرور روی پکت‌های خروجی می‌گذارد.
- `--peer-spoof-ip` — IP جعلی که این سرور از طرف مقابل **انتظار** دارد (در فیلتر BPF استفاده می‌شود).
- `--send-transport udp` / `--recv-transport tcp` — خروجی UDP، ورودی از ایران به‌صورت TCP. **باید با Transport های `local` آینه‌ای باشد.**
- `--spoof-port 8090` — پورت مبدا جعلی روی پکت‌های خروجی.

### ۶.۳ جلوگیری از لو رفتن کرنل

کرنل را از پاسخ دادن به پکت‌های اسپوف‌شده باز دارید تا IP واقعی لو نرود:

```bash
# حذف پاسخ‌های Unreachable کرنل
sudo iptables -A OUTPUT -p icmp --icmp-type destination-unreachable -j DROP

# اگر recv-transport=tcp، RST خروجی کرنل را روی پورت listen دراپ کنید
sudo iptables -A OUTPUT -p tcp --tcp-flags RST RST --sport 8090 -j DROP

# اجازه ورود ترافیک تونل
sudo iptables -A INPUT -p tcp --dport 8090 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 8090 -j ACCEPT

# ذخیره دائمی
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

---

## ۷. راه‌اندازی سرور ایران

### ۷.۱ اجرای `spoof local`

```bash
sudo spoof local \
  --listen 127.0.0.1:5000 \
  --recv-port 5001 \
  --remote <FOREIGN_REAL_IP> \
  --remote-port 8090 \
  --spoof-ip <Client_Spoof_IP> \
  --peer-spoof-ip <Server_Spoof_IP> \
  --send-transport tcp \
  --recv-transport udp \
  --spoof-port 443
```

نقش هر flag:
- `--listen 127.0.0.1:5000` — سوکت UDP که VPN client محلی به آن وصل می‌شود.
- `--recv-port 5001` — پورت دریافت پاسخ از سرور خارج.
- `--remote` / `--remote-port` — IP و پورت واقعی سرور خارج.
- `--spoof-ip` / `--peer-spoof-ip` — IP‌های جعلی (آینه‌ی flag های سمت خارج).
- `--send-transport tcp` / `--recv-transport udp` — خروجی TCP، ورودی UDP. **باید با سمت خارج تطابق داشته باشد.**
- `--spoof-port 443` — پورت مبدا جعلی. پورت `443` ترافیک خروجی را شبیه HTTPS می‌کند.

### ۷.۲ جلوگیری از لو رفتن کرنل

```bash
sudo iptables -A OUTPUT -p icmp --icmp-type destination-unreachable -j DROP

# اگر send-transport=tcp، کرنل سعی می‌کند RST بفرستد چون سوکتی روی مبدا جعلی باز نیست.
# آن‌ها را دراپ کنید:
sudo iptables -A OUTPUT -p tcp --tcp-flags RST RST -d <FOREIGN_REAL_IP> -j DROP

sudo netfilter-persistent save
```

---

## ۸. اتصال VPN Client

اکنون یک «نقطه پایانی VPN» محلی روی `127.0.0.1:5000` در سرور ایران دارید. WireGuard client را به آن متصل کنید.

### ۸.۱ WireGuard Client روی خود سرور ایران

`/etc/wireguard/wg0.conf` روی سرور ایران:
```ini
[Interface]
Address = 10.66.66.2/24
PrivateKey = <محتوای_/etc/wireguard/client.key>
DNS = 1.1.1.1, 8.8.8.8

[Peer]
PublicKey = <PUBLIC_KEY_سرور_خارج>
Endpoint = 127.0.0.1:5000
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

اجرا:
```bash
sudo systemctl enable --now wg-quick@wg0
```

تست:
```bash
curl --interface wg0 https://ifconfig.me
# باید FOREIGN_REAL_IP را برگرداند
```

### ۸.۲ استفاده از WireGuard به‌عنوان Upstream برای پنل پروکسی

اگر روی سرور ایران پنل Xray / Sing-box / Marzban / 3x-ui دارید، Outbound آن را از طریق interface `wg0` Route کنید تا کاربران نهایی ترافیک Route شده از خارج را از طریق تونل اسپوف بگیرند.

در Xray:
```json
{
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "via-wg",
      "sendThrough": "10.66.66.2"
    }
  ]
}
```

---

## ۹. اجرا به‌صورت سرویس systemd

### سرور خارج — `/etc/systemd/system/spoof-remote.service`

```ini
[Unit]
Description=Spoof Tunnel (remote)
After=network-online.target wg-quick@wg0.service
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/spoof remote \
  --listen-port 8090 \
  --forward 127.0.0.1:51820 \
  --client-ip IRAN_REAL_IP \
  --client-port 5001 \
  --spoof-ip Server_Spoof_IP \
  --peer-spoof-ip Client_Spoof_IP \
  --send-transport udp \
  --recv-transport tcp \
  --spoof-port 8090
Restart=always
RestartSec=3
LimitNOFILE=1048576
AmbientCapabilities=CAP_NET_RAW CAP_NET_ADMIN
CapabilityBoundingSet=CAP_NET_RAW CAP_NET_ADMIN

[Install]
WantedBy=multi-user.target
```

### سرور ایران — `/etc/systemd/system/spoof-local.service`

```ini
[Unit]
Description=Spoof Tunnel (local)
After=network-online.target
Wants=network-online.target
Before=wg-quick@wg0.service

[Service]
ExecStart=/usr/local/bin/spoof local \
  --listen 127.0.0.1:5000 \
  --recv-port 5001 \
  --remote FOREIGN_REAL_IP \
  --remote-port 8090 \
  --spoof-ip Client_Spoof_IP \
  --peer-spoof-ip Server_Spoof_IP \
  --send-transport tcp \
  --recv-transport udp \
  --spoof-port 443
Restart=always
RestartSec=3
LimitNOFILE=1048576
AmbientCapabilities=CAP_NET_RAW CAP_NET_ADMIN
CapabilityBoundingSet=CAP_NET_RAW CAP_NET_ADMIN

[Install]
WantedBy=multi-user.target
```

فعال‌سازی هر دو:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now spoof-remote   # سرور خارج
sudo systemctl enable --now spoof-local    # سرور ایران

# نمایش لاگ
journalctl -u spoof-remote -f
journalctl -u spoof-local -f
```

---

## ۱۰. حالت‌های Transport و تنظیم

### ۱۰.۱ انتخاب Transport

`send-transport` و `recv-transport` می‌توانند `tcp`، `udp`، `icmp` یا `icmpv6` باشند. می‌توانند در هر جهت متفاوت باشند (و **باید بین دو سرور آینه‌ای باشند**).

| سناریو | ایران `send` / `recv` | خارج `send` / `recv` |
|--------|------------------------|----------------------|
| پیش‌فرض (TCP بالا، UDP پایین) | `tcp` / `udp` | `udp` / `tcp` |
| همه UDP | `udp` / `udp` | `udp` / `udp` |
| فقط ICMP (شبیه پینگ) | `icmp` / `icmp` | `icmp` / `icmp` |
| TCP در هر دو جهت | `tcp` / `tcp` | `tcp` / `tcp` |

قاعده آینه‌ای: **`send-transport` ایران = `recv-transport` خارج** و **`recv-transport` ایران = `send-transport` خارج**.

### ۱۰.۲ Spoof Port

`--spoof-port` (پیش‌فرض `443` در `local` و `8090` در `remote`) پورت مبدا جعلی روی پکت‌های خروجی است. `443` آن را شبیه HTTPS و `53` شبیه DNS می‌کند. چیزی انتخاب کنید که فایروال خروجی شما فیلترش نکند.

### ۱۰.۳ MTU

حمل با اسپوف هیچ حاشیه‌ای برای فرگمنت شدن نمی‌گذارد. اگر انتقال‌های بزرگ متوقف می‌شوند، MTU روی WireGuard را پایین بیاورید:
```ini
[Interface]
MTU = 1280
```

---

## ۱۱. چرخش IP اسپوف

به‌جای `--spoof-ip` می‌توانید لیستی از IP‌ها بدهید که تونل به‌صورت round-robin بین آن‌ها بچرخد:

```bash
# /etc/spoof/spoof-ips.txt
203.0.113.50
203.0.113.51
203.0.113.52
198.51.100.7
```

سپس:
```bash
spoof local --spoof-ip-file /etc/spoof/spoof-ips.txt ...
```

این کار تشخیص الگو را سخت‌تر می‌کند و می‌توانید بار را بین چند «هویت» پخش کنید.

---

## ۱۲. فایل کانفیگ (جایگزین CLI)

هم `spoof local` و هم `spoof remote` پارامتر `-c <مسیر>` می‌گیرند، و `spoof run -c <مسیر>` mode را از خود کانفیگ تشخیص می‌دهد. کلیدها معادل flag های CLI هستند (با `_` به‌جای `-`).

نمونه برای `local`:
```json
{
  "mode": "local",
  "listen": "127.0.0.1:5000",
  "recv_port": 5001,
  "remote": "FOREIGN_REAL_IP",
  "remote_port": 8090,
  "spoof_ip": "Client_Spoof_IP",
  "peer_spoof_ip": "Server_Spoof_IP",
  "send_transport": "tcp",
  "recv_transport": "udp",
  "spoof_port": 443
}
```

نمونه برای `remote`:
```json
{
  "mode": "remote",
  "listen_port": 8090,
  "forward": "127.0.0.1:51820",
  "client_ip": "IRAN_REAL_IP",
  "client_port": 5001,
  "spoof_ip": "Server_Spoof_IP",
  "peer_spoof_ip": "Client_Spoof_IP",
  "send_transport": "udp",
  "recv_transport": "tcp",
  "spoof_port": 8090
}
```

flag های CLI بر مقادیر کانفیگ اولویت دارند. اجرا با:
```bash
sudo spoof run -c /etc/spoof/config.json
```

---

## ۱۳. مرجع کامل CLI

### `spoof local` — سمت ایران (نقطه ورود)
| Flag | پیش‌فرض | توضیح |
|------|---------|-------|
| `-c, --config` | — | مسیر فایل کانفیگ (flag های CLI اولویت دارند) |
| `-l, --listen` | `127.0.0.1:5000` | آدرس UDP که برنامه/VPN client به آن وصل می‌شود |
| `--peer-spoof-ip` | — | Source IP مورد انتظار از پکت‌های ورودی سرور |
| `--recv-port` | `5001` | پورت دریافت پکت از سرور |
| `--recv-transport` | `udp` | Transport دریافت: `tcp`, `udp`, `icmp`, `icmpv6` |
| `-r, --remote` | — | IP واقعی سرور خارج |
| `-p, --remote-port` | `8090` | پورت سرور خارج |
| `--send-transport` | `tcp` | Transport ارسال: `tcp`, `udp`, `icmp`, `icmpv6` |
| `-s, --spoof-ip` | — | یک IP جعلی |
| `--spoof-ip-file` | — | فایل لیست IP جعلی (هر خط یکی، round-robin) |
| `--spoof-port` | `443` | پورت مبدا جعلی |

### `spoof remote` — سمت خارج (نقطه خروج)
| Flag | پیش‌فرض | توضیح |
|------|---------|-------|
| `-c, --config` | — | مسیر فایل کانفیگ |
| `--client-ip` | — | IP واقعی کلاینت ایران |
| `--client-port` | `5001` | پورت دریافت کلاینت ایران |
| `-f, --forward` | `127.0.0.1:51820` | مقصد UDP برای forward پکت‌های decapsulate شده |
| `-l, --listen-port` | `8090` | پورت دریافت پکت‌های تونل |
| `--peer-spoof-ip` | — | Source IP مورد انتظار از پکت‌های ورودی کلاینت |
| `--recv-transport` | `tcp` | Transport دریافت |
| `--send-transport` | `udp` | Transport ارسال |
| `-s, --spoof-ip` | — | یک IP جعلی |
| `--spoof-ip-file` | — | فایل لیست IP جعلی (round-robin) |
| `--spoof-port` | `8090` | پورت مبدا جعلی |

### `spoof run` — حالت کانفیگ‌محور
| Flag | پیش‌فرض | توضیح |
|------|---------|-------|
| `-c, --config` | `config.json` | مسیر فایل کانفیگ |

### `spoof completion <shell>`
اسکریپت autocompletion برای `bash`, `zsh`, `fish` یا `powershell` می‌سازد:
```bash
spoof completion bash | sudo tee /etc/bash_completion.d/spoof
```

---

## ۱۴. عیب‌یابی

| علامت | علت محتمل | راه‌حل |
|------|----------|--------|
| `cannot open raw socket: permission denied` | عدم دسترسی root/cap | با `sudo` اجرا کنید یا `CAP_NET_RAW + CAP_NET_ADMIN` بدهید |
| تونل بالا می‌آید اما هندشیک WireGuard کامل نمی‌شود | پکت اسپوف عبور نمی‌کند | بخش ۳ را در **هر دو جهت** تکرار کنید؛ مطمئن شوید `--peer-spoof-ip` برابر با `--spoof-ip` **سمت مقابل** است |
| WireGuard هندشیک می‌کند اما `curl` روی `wg0` گیر می‌کند | کرنل به peer دارد RST/Unreachable می‌فرستد | قوانین OUTPUT DROP بخش‌های ۶.۳ و ۷.۲ را بزنید |
| موقتاً کار می‌کند، بعد قطع می‌شود | NAT timeout مسیر | `PersistentKeepalive = 25` در WireGuard |
| انتقال‌های بزرگ متوقف می‌شوند | فرگمنت IP | `MTU` در WireGuard را روی `1280` بگذارید |
| لاگ کرنل پیام `martian source` می‌دهد | طبیعی و بی‌خطر — کرنل پکتی می‌بیند که Source IP اش با interface ورودی نمی‌خواند | اختیاراً با `sudo sysctl -w net.ipv4.conf.all.log_martians=0` خاموش کنید |
| خطای transport mismatch | `send-transport` ایران ≠ `recv-transport` خارج (یا برعکس) | بخش ۱۰.۱ آینه‌ای کنید |

نمایش لاگ: `journalctl -u spoof-local -f` یا `journalctl -u spoof-remote -f`.

---

## ۱۵. نکات امنیتی

- **اسپوف تانل رمزنگاری ندارد.** حتماً روی آن یک VPN واقعی (WireGuard / OpenVPN / Hysteria) راه بیندازید، وگرنه ترافیک رمزنشده عبور می‌کند.
- جعل IP ممکن است در قوانین کشور یا قرارداد پروایدر شما غیرمجاز باشد. با مسئولیت خودتان استفاده کنید.
- این ابزار صرفاً برای حفظ دسترسی به اینترنت در شرایط قطعی‌های دولتی طراحی شده است؛ آن را علیه اشخاص ثالث استفاده نکنید.

---

## ۱۶. لایسنس

Apache 2.0 — فایل [LICENSE](./LICENSE) را ببینید.

---

> توسعه داده شده و تست شده در قطعی کامل اینترنت ایران پس از قیام خونین ۱۸ و ۱۹ دی ۱۴۰۴.
