# 🎮 بازی آنلاین واقعی (Real-Time Multiplayer)

این بخش از پروژه امکان بازی **واقعی بین دو نفر در دو گوشی/دستگاه جداگانه** را فراهم می‌کند.

## 🏗️ معماری (بهترین روش: WebSocket)

```
بازیکن ۱ (گوشی)                    بازیکن ۲ (گوشی)
     │                                    │
     └──────── WebSocket (لحظه‌ای) ────────┘
                      │
                      ▼
              🖥️ سرور realtime.py
         ┌──────────────────────────┐
         │  • Matchmaking (تصادفی)  │
         │  • Challenge (چالش)      │
         │  • منطق بازی (سرور)      │
         │  • WebSocket broadcast   │
         └──────────────────────────┘
                      │
                      ▼
                📦 SQLite
```

## ✅ چرا WebSocket؟ (بهترین روش)

| روش | تاخیر | مناسب بازی؟ |
|------|--------|-------------|
| HTTP Polling | ۱-۲ ثانیه | ❌ کند |
| **WebSocket** | **< ۵۰ms** | **✅ ایده‌آل** |
| Server-Sent Events | یک‌طرفه | ⚠️ محدود |

WebSocket ارتباط **دوطرفه و لحظه‌ای** برقرار می‌کند — حرکت هر بازیکن فوراً (کمتر از ۵۰ms) برای حریف نمایش داده می‌شود.

## 🚀 راه‌اندازی

### ۱. اجرای سرور
```bash
cd server
pip install -r requirements.txt
python realtime.py
```
سرور روی پورت ۸۰۸۰ اجرا می‌شود:
- WebSocket: `ws://localhost:8080/ws`
- HTTP API: `http://localhost:8080`

### ۲. تنظیم فرانت‌اند
فایل `.env` بسازید:
```
VITE_WS_URL=ws://YOUR_SERVER_IP:8080/ws
VITE_API_URL=http://YOUR_SERVER_IP:8080
```

### ۳. اجرای فرانت‌اند
```bash
npm run dev
```

## 🎯 سه روش بازی

### ۱. 🎲 بازی تصادفی (Matchmaking)
- بازیکن بازی را انتخاب می‌کند
- روی «شروع جستجوی حریف» می‌زند
- سرور **به‌صورت تصادفی** او را با یک بازیکن دیگر در صف جفت می‌کند
- بازی فوراً شروع می‌شود

### ۲. ⚔️ چالش دوست (Challenge)
- لیست بازیکنان **آنلاین** نمایش داده می‌شود
- بازیکن روی «چالش» می‌زند
- حریف یک **modal چالش** دریافت می‌کند
- با «قبول» کردن، بازی شروع می‌شود

### ۳. 🔗 کد اتاق (Room Code)
- (روش قبلی - در api.py موجود)

## 📡 پروتکل WebSocket

### Client → Server
```json
{"type": "auth", "user_id": 123456, "name": "علی"}
{"type": "queue_join", "game_type": "tictactoe"}
{"type": "queue_leave"}
{"type": "challenge_send", "target_id": 789, "game_type": "rps"}
{"type": "challenge_respond", "challenge_id": "ch_123", "accept": true}
{"type": "move", "game_id": 1, "action": "ttt_move", "payload": {"position": 4}}
{"type": "game_leave", "game_id": 1}
```

### Server → Client
```json
{"type": "auth_ok", "players_online": 5}
{"type": "queue_joined", "position": 1}
{"type": "match_found", "game_id": 1, "opponent_name": "رضا", "player_num": 1}
{"type": "game_state", "state": {...}}
{"type": "challenge_received", "from_name": "سارا", "game_type": "rps"}
{"type": "challenge_declined"}
```

## 🛡️ امنیت (Server-Authoritative)

تمام منطق بازی **روی سرور** اجرا می‌شود:
- ✅ امکان تقلب = صفر
- ✅ امتیازها در SQLite ذخیره می‌شوند
- ✅ انتقال امتیاز فقط پس از پایان بازی معتبر
- ✅ ثبت‌نام کاربران با آیدی عددی بله

## 💰 جریان امتیاز

```
بازی شروع می‌شود (هر نفر ۱۰ امتیاز شرط)
        ↓
بازی روی سرور اجرا می‌شود
        ↓
برنده مشخص شد:
  • ۱۰ امتیاز از بازنده کسر می‌شود
  • ۱۰ امتیاز به برنده اضافه می‌شود
  • تراکنش در SQLite ثبت می‌شود
```

## 🔌 اتصال به بله

در Mini App واقعی بله، آیدی کاربر از `initData` بله دریافت می‌شود:
```javascript
// Bale Mini App
const initData = window.Telegram?.WebApp?.initDataUnsafe;
const userId = initData?.user?.id;  // آیدی عددی واقعی بله
const userName = initData?.user?.first_name;
```

این آیدی به جای آیدی تصادفی فعلی استفاده می‌شود تا کاربران واقعی بله شناسایی شوند.
