# 🚀 راهنمای استقرار پروژه روی سرور لینوکس (Ubuntu / Debian)

این پروژه از دو بخش اصلی تشکیل شده است:
1. **فرانت‌اند (Mini App React):** به صورت یک فایل وب کاملاً بهینه کامپایل می‌شود و از طریق Nginx سرویس داده می‌شود.
2. **بک‌اند (Python Realtime WebSocket Server):** موتور بلادرنگ چندنفره، پردازش بازی‌ها و دیتابیس SQLite.

---

## 🛠️ مرحله ۱: آماده‌سازی سرور لینوکس

ابتدا بسته‌های پایه‌ای سرور را آپدیت و ابزارهای مورد نیاز (Python 3.12، Git، Nginx و Node.js) را نصب کنید:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-pip python3-venv nginx git curl software-properties-common

# نصب Node.js (برای بیلد فرانت‌اند)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

---

## 📦 مرحله ۲: دریافت کد و بیلد فرانت‌اند

کد پروژه را روی سرور منتقل کرده و فرانت‌اند را بیلد کنید:

```bash
# وارد دایرکتوری پروژه شوید
cd /var/www/bale-game-center

# نصب وابستگی‌های Node.js و ساخت خروجی Single File
npm install
npm run build
```

پس از اجرای موفق `npm run build`، پوشه `dist/` ایجاد می‌شود که حاوی فایل `index.html` بهینه است.

---

## 🐍 مرحله ۳: راه‌اندازی سرور پایتون (Realtime Server)

برای بک‌اند، یک محیط مجازی پایتون (virtualenv) ایجاد کنید و وابستگی‌ها را نصب نمایید:

```bash
cd /var/www/bale-game-center/server

# ساخت و فعال‌سازی محیط مجازی
python3 -m venv venv
source venv/bin/activate

# نصب کتابخانه‌ها
pip install --upgrade pip
pip install -r requirements.txt
```

فایل تنظیمات محیطی `.env` را در دایرکتوری `server/` بسازید:

```bash
nano /var/www/bale-game-center/server/.env
```

محتوای زیر را درون آن قرار دهید:

```env
BALE_BOT_TOKEN="123456789:abcdefghijklmnopqrstuvwxyz"
BOT_SECRET_KEY="your-random-secure-secret-key-32-chars"
ALLOWED_ORIGINS="https://game.yourdomain.com"
PORT=8080
```

---

## ⚙️ مرحله ۴: اجرای خودکار بک‌اند با Systemd Service

برای اینکه سرور پایتون در پس‌زمینه اجرا شود و در صورت ریستارت سرور خودکار روشن شود، یک سرویس Systemd بسازید:

```bash
sudo nano /etc/systemd/system/bale-game.service
```

محتوای زیر را درون فایل بگذارید:

```ini
[Unit]
Description=Bale Game Center Realtime WebSocket Server
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/var/www/bale-game-center/server
Environment="PATH=/var/www/bale-game-center/server/venv/bin"
ExecStart=/var/www/bale-game-center/server/venv/bin/python realtime.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

مجوز پوشه را به `www-data` بدهید و سرویس را فعال کنید:

```bash
sudo chown -R www-data:www-data /var/www/bale-game-center
sudo systemctl daemon-reload
sudo systemctl enable bale-game
sudo systemctl start bale-game

# بررسی وضعیت اجرا
sudo systemctl status bale-game
```

---

## 🌐 مرحله ۵: تنظیم Nginx به عنوان Reverse Proxy و وب‌سرور

فایل تنظیمات Nginx برای دامنه خود (مثلاً `game.yourdomain.com`) را بسازید:

```bash
sudo nano /etc/nginx/sites-available/bale-game
```

محتوای زیر را قرار دهید:

```nginx
server {
    listen 80;
    server_name game.yourdomain.com;

    # سرویس دادن فایل‌های فرانت‌اند
    root /var/www/bale-game-center/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # هدایت درخواست‌های WebSocket به سرور پایتون
    location /ws {
        proxy_pass http://127.0.0.1:8080/ws;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }

    # هدایت API های HTTP به سرور پایتون
    location /api/ {
        proxy_pass http://127.0.0.1:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

سایت را فعال و Nginx را ریستارت کنید:

```bash
sudo ln -s /etc/nginx/sites-available/bale-game /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## 🔒 مرحله ۶: فعال‌سازی گواهی رایگان SSL (HTTPS) با Certbot

از آنجا که مینی‌اپ‌های بله باید حتماً روی بستر **HTTPS** باشند:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d game.yourdomain.com
```

اکنون آدرس وب‌سایت شما (`https://game.yourdomain.com`) آماده اتصال به BotFather بله به عنوان Mini App است! 🎮✨
