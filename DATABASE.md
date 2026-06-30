# 🗄️ دیتابیس SQLite حرفه‌ای

تمام عملیات ربات در **SQLite** ثبت و نگهداری می‌شود. هیچ چیزی از دست نمی‌رود.

## 📊 ساختار جداول

### ۱. `users` — کاربران
| ستون | توضیح |
|------|-------|
| `user_id` | شناسه عددی یکتای بله |
| `username` | نام کاربری بله |
| `first_name` / `last_name` | نام واقعی |
| `phone` | شماره موبایل |
| `points` | موجودی امتیاز فعلی |
| `total_earned` | کل امتیاز برده‌شده |
| `total_spent` | کل امتیاز باخته‌شده |
| `total_deposited` | کل شارژ از پاکت |
| `total_withdrawn` | کل برداشت |
| `wins` / `losses` / `draws` | آمار بازی‌ها |
| `games_played` | تعداد کل بازی‌ها |
| `streak_days` | استریک روزانه |
| `last_daily` | آخرین دریافت جایزه روزانه |
| `status` | وضعیت کاربر |
| `created_at` / `updated_at` | زمان ثبت‌نام و به‌روزرسانی |

### ۲. `games` — تاریخچه بازی‌ها
| ستون | توضیح |
|------|-------|
| `id` | شناسه بازی |
| `game_type` | نوع بازی (دوز، چهاردرخط، ...) |
| `player1_id` / `player2_id` | آیدی دو بازیکن |
| `stake` | مبلغ شرط |
| `winner_id` | آیدی برنده (NULL = مساوی) |
| `status` | waiting/active/finished/cancelled |
| `board_state` | وضعیت نهایی برد (JSON) |
| `moves_count` | تعداد حرکات |
| `started_at` / `finished_at` | زمان شروع و پایان |
| `duration_sec` | مدت زمان بازی (ثانیه) |
| `room_code` | کد اتاق (در صورت داشتن) |
| `created_at` | زمان ساخت بازی |

### ۳. `moves` — تمام حرکات بازیکنان
هر حرکت هر بازیکن در این جدول ثبت می‌شود:
- `game_id`, `user_id`, `player_num`
- `move_type`, `move_data` (JSON کامل حرکت)
- `move_number` (شماره حرکت)
- `created_at`

### ۴. `transactions` — دفتر کل مالی
| ستون | توضیح |
|------|-------|
| `user_id` | کاربر |
| `amount` | مبلغ (+ درآمد / - هزینه) |
| `balance_before` | موجودی قبل از تراکنش |
| `balance_after` | موجودی بعد از تراکنش |
| `tx_type` | نوع: win, loss, draw, deposit, withdrawal, bot_fee, daily_reward |
| `reference_id` / `reference_type` | ارجاع به بازی یا پرداخت |
| `description` | توضیح فارسی |
| `metadata` | داده اضافی JSON |
| `created_at` | زمان تراکنش |

### ۵. `payments` — پرداخت‌های پاکت بله
تمام فاکتورهای `sendInvoice` در این جدول ثبت می‌شوند:
- `amount_toman`, `points_added`
- `provider_token`, `payload`
- `status`: pending / completed
- `verified_at`

### ۶. `daily_rewards` — جایزه‌های روزانه
هر دریافت جایزه روزانه با روز و امتیاز ثبت می‌شود.

### ۷. `queue` — صف matchmaking
کاربران در انتظار حریف.

### ۸. `audit_log` — لاگ رویدادها
- `connected` / `disconnected`
- `queue_join` / `queue_leave`
- `game_started` / `resigned`
- `challenge_sent` / `challenge_respond`
- `daily_claim`

## ✅ تراکنش‌های اتمی (Atomic)

تمام انتقال‌های امتیاز در **یک تراکنش دیتابیس** انجام می‌شوند:

```python
BEGIN TRANSACTION
  1. SELECT points FROM users (winner + loser) FOR UPDATE
  2. UPDATE winner SET points = points + stake
  3. UPDATE loser  SET points = points - stake
  4. INSERT INTO transactions (winner, +stake)
  5. INSERT INTO transactions (loser,  -stake)
  6. UPDATE users SET wins/losses
COMMIT
```

اگر هر یک از مراحل ناموفق شود، هیچ‌کدام اعمال نمی‌شود (rollback).

## 🔄 جریان کامل یک بازی واقعی

```
۱. کاربر وارد می‌شود
   → INSERT/UPDATE users (اگر وجود نداشت)
   → INSERT audit_log (connected)

۲. وارد صف matchmaking می‌شود
   → INSERT queue
   → INSERT audit_log (queue_join)

۳. حریف پیدا می‌شود
   → DELETE queue (هر دو)
   → INSERT games (status='active')
   → INSERT audit_log (game_started) × ۲

۴. هر حرکت بازیکن
   → INSERT moves
   → UPDATE games SET board_state, moves_count

۵. بازی تمام می‌شود
   → UPDATE games SET winner_id, status='finished', finished_at
   → UPDATE users (points, wins, losses)
   → INSERT transactions (win) برای برنده
   → INSERT transactions (loss) برای بازنده
   → INSERT audit_log (game_finished)

۶. خروج کاربر
   → UPDATE audit_log (disconnected)
```

## 🔍 مشاهده داده‌ها

می‌توانید مستقیماً با SQLite Browser یا دستور `sqlite3` داده‌ها را ببینید:

```bash
sqlite3 server/bale_game_bot.db

# تعداد کاربران
SELECT COUNT(*) FROM users;

# تعداد بازی‌های تمام‌شده
SELECT COUNT(*) FROM games WHERE status='finished';

# آخرین تراکنش‌ها
SELECT * FROM transactions ORDER BY created_at DESC LIMIT 10;

# آمار یک کاربر
SELECT * FROM users WHERE user_id = 123456;

# تاریخچه بازی‌های یک کاربر
SELECT * FROM v_game_history
WHERE player1_id = 123456 OR player2_id = 123456;
```

## 📈 View-های آماده

- `v_game_history`: بازی‌ها با نام بازیکنان و برنده
- `v_financial_summary`: خلاصه درآمد/هزینه هر کاربر

## 🚀 API Endpoints برای مشاهده داده‌ها

| Endpoint | توضیح |
|----------|-------|
| `GET /user/{id}` | اطلاعات کاربر |
| `GET /transactions/{id}` | تراکنش‌های کاربر |
| `GET /history/{id}` | تاریخچه بازی‌های کاربر |
| `GET /moves/{game_id}` | حرکات یک بازی |
| `GET /leaderboard` | جدول امتیازات |
| `GET /stats` | آمار کلی سرور |
| `GET /online` | لیست کاربران آنلاین |
