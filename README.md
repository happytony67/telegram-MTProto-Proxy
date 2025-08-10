# MTProto Proxy Auto-Installer (Docker) نصب سریع و آسان
اسکریپت نصب خودکار پروکسی MTProto با Docker که در پایان **فقط لینک اتصال تلگرام** (`tg://...`) را چاپ می‌کند.  
روی اوبونتو/دبیان/سنتر‌اواس/آر‌اچ‌ئی‌ال و مشتقاتشان تست شده است.

## ویژگی‌ها
- نصب خودکار Docker (در صورت نبود)
- باز کردن پورت در UFW/Firewalld (در صورت وجود)
- تولید خودکار **SECRET** شانسی 16 بایتی (32 هگز)
- اجرای کانتینر رسمی `telegrammessenger/proxy`
- چاپ فقط یک خروجی نهایی: `tg://proxy?server=...&port=...&secret=...`

## پیش‌نیازها
- دسترسی `root` یا `sudo`
- یک سرور لینوکسی با IP عمومی
- پورت پیش‌فرض `443` (قابل تغییر)
اگر روی پورت ۴۴٣ کار نکرد روی پورت ٨۴۴٣ بگذارید  

## راه‌اندازی سریع
1) فایل اسکریپت را روی سرور بسازید:
```bash
nano install_mtproto.sh
````

2. محتوای کد اصلی را از کپی و داخل فایلی که باز کرده‌اید ذخیره کنید (Ctrl+O سپس Enter، و خروج با Ctrl+X).


## کد اصلی


````
#!/usr/bin/env bash
# MTProto Proxy auto-install via Docker, prints ONLY the tg:// link
# Usage: sudo bash install_mtproto.sh [PORT]
set -e

PORT="${1:-443}"

# Quiet helpers
qrun() { "$@" >/dev/null 2>&1 || true; }

# Ensure basic tools
if command -v apt-get >/dev/null 2>&1; then
  qrun apt-get update
  qrun apt-get install -y docker.io curl ca-certificates gnupg lsb-release
elif command -v dnf >/dev/null 2>&1; then
  qrun dnf install -y docker curl
elif command -v yum >/dev/null 2>&1; then
  qrun yum install -y docker curl
fi

# Start/enable Docker
qrun systemctl enable docker
qrun systemctl start docker
if ! docker info >/dev/null 2>&1; then
  # Fallback try
  qrun service docker start
fi

# Open firewall if present
qrun ufw allow "${PORT}"/tcp
if command -v firewall-cmd >/dev/null 2>&1; then
  qrun firewall-cmd --permanent --add-port="${PORT}"/tcp
  qrun firewall-cmd --reload
fi

# Generate 16-byte (32 hex) secret
if command -v xxd >/dev/null 2>&1; then
  SECRET="$(head -c 16 /dev/urandom | xxd -p -c 256)"
else
  # Portable hex generator
  SECRET="$(head -c 16 /dev/urandom | od -An -tx1 | tr -d ' \n')"
fi

# Pull and run container
qrun docker rm -f mtp
qrun docker pull telegrammessenger/proxy:latest
qrun docker run -d --name mtp --restart=always -p "${PORT}:443" -e SECRET="${SECRET}" telegrammessenger/proxy:latest

# Detect public IP
detect_ip() {
  IP=""
  if command -v dig >/dev/null 2>&1; then
    IP="$(dig +short myip.opendns.com @resolver1.opendns.com 2>/dev/null | tail -n1)"
  fi
  if [ -z "$IP" ] && command -v curl >/dev/null 2>&1; then
    IP="$(curl -fsS ifconfig.me 2>/dev/null || true)"
  fi
  if [ -z "$IP" ] && command -v curl >/dev/null 2>&1; then
    IP="$(curl -fsS ipinfo.io/ip 2>/dev/null || true)"
  fi
  if [ -z "$IP" ]; then
    # last resort: inspect default interface
    IFACE="$(ip route 2>/dev/null | awk '/default/ {print $5; exit}')"
    IP="$(ip -4 addr show "$IFACE" 2>/dev/null | awk '/inet / {print $2}' | cut -d/ -f1 | head -n1)"
  fi
  echo "$IP"
}
IP="$(detect_ip)"

# Print ONLY the tg:// link
printf "tg://proxy?server=%s&port=%s&secret=%s\n" "$IP" "$PORT" "$SECRET"
````


3. اجرا:

```bash
chmod +x install_mtproto.sh
sudo bash install_mtproto.sh
# یا با پورت دلخواه (مثلاً 8443):
# sudo bash install_mtproto.sh 8443
```

در پایان، فقط یک لینک مثل زیر چاپ می‌شود. همان را در تلگرام اضافه کنید:

```
tg://proxy?server=YOUR_IP&port=443&secret=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

## بروزرسانی یا تغییر پورت

* توقف و حذف کانتینر:

```bash
sudo docker rm -f mtp
```

* اجرای مجدد اسکریپت با پورت جدید:

```bash
sudo bash install_mtproto.sh 8443
```

## حذف کامل

```bash
sudo docker rm -f mtp
sudo systemctl stop docker || true
```

## عیب‌یابی

* **Docker بالا نمی‌آید:** لاگ سرویس را چک کنید:

```bash
sudo journalctl -u docker --no-pager | tail -n 200
```

* **پورت بسته است:** فایروال را بررسی کنید (`ufw status` یا `firewall-cmd --list-ports`) و پورت انتخابی را باز کنید.
* **IP اشتباه/خصوصی چاپ می‌شود:** اگر پشت NAT هستید، آدرس عمومی را دستی جایگزین کنید.
* **فایل‌های بزرگ/پورت 443 درگیر:** پورت دیگری مثل `8443` یا `2053` را امتحان کنید.

## نکات امنیتی و حقوقی

* SECRET را خصوصی نگه دارید.
* استفاده از پروکسی تابع قوانین محلی شماست؛ مسئولیت استفاده با کاربر است.


