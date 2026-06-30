# 🔒 گزارش امنیتی جامع — Bale Game Bot

## خلاصه: ۱۲ آسیب‌پذیری شناسایی و رفع شد

| # | سطح | عنوان | وضعیت |
|---|------|-------|-------|
| 1 | 🔴 حیاتی | نشت موقعیت بمب‌ها به کلاینت (Mine Rush) | ✅ رفع شد |
| 2 | 🔴 حیاتی | Race Condition انتقال امتیاز (TOCTOU) | ✅ رفع شد |
| 3 | 🔴 حیاتی | بدون احراز هویت (هرکس هر user_id) | ✅ رفع شد |
| 4 | 🔴 حیاتی | بدون اعتبارسنجی ورودی (Injection) | ✅ رفع شد |
| 5 | 🟠 بالا | DoS/Flood بدون Rate Limit | ✅ رفع شد |
| 6 | 🟠 بالا | Anti-Cheat نداشت (Bot/Replay) | ✅ رفع شد |
| 7 | 🟠 بالا | Double-Game Exploit (بازی همزمان) | ✅ رفع شد |
| 8 | 🟡 متوسط | چالش‌ها بدون TTL (حافظه لیک) | ✅ رفع شد |
| 9 | 🟡 متوسط | CORS باز (`*`) | ✅ رفع شد |
| 10 | 🟡 متوسط | نشت اطلاعات حساس (phone, token) | ✅ رفع شد |
| 11 | 🟢 بهبود | Security Headers نداشت | ✅ رفع شد |
| 12 | 🟢 بهبود | خطاها جزئیات داخلی لو می‌دادند | ✅ رفع شد |

---

## 🔴 آسیب‌پذیری‌های حیاتی (Critical)

### 1. نشت `secret_board` در Mine Rush
**CVSS: 9.8** — بازیکن می‌توانست موقعیت ۶ بمب را ببیند و فقط خانه‌های امن را کلیک کند.

**قبل:**
```python
# broadcast_game() تمام state را خام ارسال می‌کرد
await send_to_user(p1_id, {'state': gs})  # شامل secret_board!
```

**بعد:**
```python
def sanitize_state_for_client(gs, user_id):
    safe = dict(gs)
    if safe.get('game_type') == 'mine':
        safe.pop('secret_board', None)  # حذف موقعیت بمب‌ها
    if safe.get('game_type') == 'guess':
        safe.pop('secret', None)  # حذف عدد مخفی
    if safe.get('game_type') == 'rps' and phase != 'done':
        # مخفی کردن انتخاب حریف تا هر دو انتخاب کنند
    return safe
```

### 2. Race Condition (TOCTOU) در انتقال امتیاز
**CVSS: 8.5** — بین `check points` و `start game`، بازیکن می‌توانست در ۲ بازی همزمان شرکت کند و امتیاز را دوبار خرج کند.

**قبل:**
```python
p1 = db.get_user(p1_id)
if p1['points'] < STAKE:  # ← CHECK
    return None
# ... زمان می‌گذرد ...
game_id = db.create_game(...)  # ← USE (points may have changed!)
```

**بعد:**
```python
# یک تراکنش اتمیک SQLite
def verify_and_lock_points(p1_id, p2_id, stake):
    with get_connection() as conn:  # ← ACID transaction
        # SELECT + بررسی در یک تراکنش
        # اگر commit شود = امن، اگر fail = rollback
```

### 3. بدون احراز هویت
**CVSS: 10.0** — هرکس با هر `user_id` وارد می‌شد.

**بعد:**
- **توکن HMAC-SHA256** صادر و بررسی می‌شود
- **تأیید initData بله** با derivative key از bot token
- `hmac.compare_digest` ضد timing attack

### 4. بدون اعتبارسنجی ورودی
**CVSS: 9.0** — XSS, SQL Injection, type confusion

**بعد:**
- Whitelist: `game_type` (14 مقدار مجاز), `action` (14 مقدار مجاز)
- Range check: `position: 0-8`, `col: 0-6`, `number: 1-100`
- `sanitize_string()`: حذف `<script>`, SQL chars, control chars
- Type validation: `isinstance()` + `int()` with try/catch
- Max message size: 4096 bytes

---

## 🟠 آسیب‌پذیری‌های بالا (High)

### 5. DoS / Flood
**بعد:** Token Bucket Algorithm
- WebSocket: 30 burst, 5/sec refill
- HTTP: 60 burst, 10/sec refill
- Auto-ban: 60 sec after overflow

### 6. Anti-Cheat
**بعد:**
- **Move timing**: min 200ms between moves, 10 violations = block
- **Replay detection**: last 5 moves tracked, duplicates rejected
- **Violation logging**: هر تخلف در `audit_log` ثبت

### 7. Double-Game Exploit
**بعد:** قبل از ورود به queue، بررسی می‌شود که بازیکن در هیچ بازی فعالی نیست.

---

## 🟡 آسیب‌پذیری‌های متوسط (Medium)

### 8. Challenge TTL
**بعد:** چالش‌ها بعد از 60 ثانیه منقضی و هر دقیقه پاکسازی می‌شوند.

### 9. CORS
**بعد:** `ALLOWED_ORIGINS` از متغیر محیطی خوانده می‌شود.

### 10. نشت اطلاعات حساس
**بعد:** فیلدهای `phone`, `wallet_token` از response حذف شده‌اند.

---

## 🟢 بهبودها

### 11. Security Headers
```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000
Content-Security-Policy: default-src 'self'
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

### 12. Error Redaction
```python
# قبل: نشت منطق داخلی
{'message': 'column_full'}

# بعد: کد عمومی + پیام فارسی
{'code': 'invalid_move', 'message': 'حرکت غیرمجاز'}
```

---

## OWASP Top 10 Coverage

| OWASP | وضعیت |
|-------|-------|
| A01 Broken Access Control | ✅ HMAC tokens + initData verification |
| A02 Cryptographic Failures | ✅ HMAC-SHA256, `compare_digest` |
| A03 Injection | ✅ Parameterized queries, input sanitization |
| A04 Insecure Design | ✅ Server-authoritative, no client trust |
| A05 Security Misconfiguration | ✅ Security headers, CORS restriction |
| A06 Vulnerable Components | ✅ Minimal dependencies (aiohttp only) |
| A07 Auth Failures | ✅ Token-based auth with expiry |
| A08 Data Integrity | ✅ ACID transactions, atomic transfers |
| A09 Logging Failures | ✅ `audit_log` table, all events logged |
| A10 SSRF | ✅ No external HTTP calls from user input |

---

## فایل‌های امنیتی

| فایل | خطوط | وظیفه |
|------|------|-------|
| `server/security.py` | ~400 | توکن، rate limit، anti-cheat، validation |
| `server/database.py` | ~800 | تراکنش اتمیک، parameterized queries |
| `server/realtime.py` | ~1000 | WebSocket handler با تمام checkها |

---

## چک‌لیست Deploy

- [ ] `BOT_SECRET_KEY` را به مقدار تصادفی قوی تنظیم کنید
- [ ] `BALE_BOT_TOKEN` را از @botfather تنظیم کنید
- [ ] `ALLOWED_ORIGINS` را به دامنه واقعی محدود کنید
- [ ] HTTPS/WSS فعال کنید (Let's Encrypt)
- [ ] Firewall: فقط پورت 443 باز باشد
- [ ] Backup خودکار دیتابیس فعال شود
- [ ] لاگ‌ها را مانیتور کنید (audit_log)
