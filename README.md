============================
MTProto Proxy - ALL-IN-ONE TXT (v2)
============================

این فایلِ متنی شامل محتوای سه فایلِ موردنیاز شماست.
برای استفاده:
1) در ریپوی گیت‌هاب خود، سه فایل جدید بسازید با نام‌های زیر:
   - README.md
   - install_mtproto.sh
   - LICENSE (اختیاری اما پیشنهاد می‌شود: MIT)
2) محتوای هر بخش زیر را دقیقا در فایل مربوطه کپی کنید.
3) اگر اسکریپت را روی سرور اجرا می‌کنید، طبق «مرحله ۲» در README عمل کنید.

---------------------------------
BEGIN FILE: README.md
---------------------------------
# MTProto Proxy Auto-Installer (Docker)
اسکریپت نصب خودکار پروکسی MTProto با Docker که در پایان **فقط لینک اتصال تلگرام** (`tg://...`) را چاپ می‌کند.  
روی اوبونتو/دبیان/سنت‌اواس/RHEL و مشابه‌ها قابل استفاده است.

## ویژگی‌ها
- نصب خودکار Docker (در صورت نبود)
- باز کردن پورت در UFW/Firewalld (در صورت وجود)
- تولید خودکار **SECRET** شانسی 16 بایتی (32 هگز)
- اجرای کانتینر رسمی `telegrammessenger/proxy`
- خروجیِ نهاییِ تک‌خطی: `tg://proxy?server=...&port=...&secret=...`

## پیش‌نیازها
- دسترسی `root` یا `sudo`
- یک سرور لینوکسی با IP عمومی
- پورت پیش‌فرض `443` (قابل تغییر)

## نصب سریع

### مرحله ۱: ساخت فایل اسکریپت و جای‌گذاری کد
در سرور:
```bash
nano install_mtproto.sh
```
کل محتوای فایل `install_mtproto.sh` (در همین بسته) را در ادیتور پیست کنید، سپس:
- ذخیره با **Ctrl+O** و Enter
- خروج با **Ctrl+X**

### مرحله ۲: اجرای اسکریپت و دریافت لینک
در ترمینال همین دستورات را بزنید:
```bash
chmod +x install_mtproto.sh
sudo bash install_mtproto.sh
# اگر می‌خواهید پورت را مشخص کنید (مثلاً 8443):
# sudo bash install_mtproto.sh 8443
```
در پایان، فقط یک لینک مانند زیر چاپ می‌شود. همان را در تلگرام اضافه کنید:
```
tg://proxy?server=YOUR_IP&port=443&secret=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

## بروزرسانی یا تغییر پورت
- توقف و حذف کانتینر:
```bash
sudo docker rm -f mtp
```
- اجرای مجدد اسکریپت با پورت جدید:
```bash
sudo bash install_mtproto.sh 8443
```

## حذف کامل
```bash
sudo docker rm -f mtp
sudo systemctl stop docker || true
```

## عیب‌یابی
- **Docker بالا نمی‌آید:** لاگ سرویس را چک کنید:
```bash
sudo journalctl -u docker --no-pager | tail -n 200
```
- **پورت بسته است:** وضعیت فایروال را بررسی کنید (`ufw status` یا `firewall-cmd --list-ports`) و پورت انتخابی را باز کنید.
- **IP غلط/خصوصی چاپ می‌شود:** اگر پشت NAT هستید، آدرس عمومی را دستی در لینک جایگزین کنید.
- **پورت 443 درگیر است:** از پورت‌های دیگر مثل `8443` یا `2053` استفاده کنید.

## نکات امنیتی و حقوقی
- SECRET را خصوصی نگه دارید.
- استفاده از پروکسی تابع قوانین محلی شماست؛ مسئولیت استفاده با کاربر است.

## مجوز
MIT (متن مجوز در فایل `LICENSE`)

---------------------------------
END FILE: README.md
---------------------------------


---------------------------------
BEGIN FILE: install_mtproto.sh
---------------------------------
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
---------------------------------
END FILE: install_mtproto.sh
---------------------------------


---------------------------------
BEGIN FILE: LICENSE (MIT)
---------------------------------
MIT License

Copyright (c)

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included
in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
---------------------------------
END FILE: LICENSE (MIT)
---------------------------------
