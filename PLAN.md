# PLAN.md — edgetunnel (نقشه توسعه)

## 📋 آنالیز پروژه

**پروژه:** `edgetunnel 2.1` — تونل لبه‌ای مبتنی بر Cloudflare Workers/Pages
**فایل اصلی:** `_worker.js` (۵۹۰۵ خط، تک‌فایل) — نسخه `2026-06-01 15:49:39`
**استقرار:** Cloudflare Workers یا Pages (`wrangler.toml` → `main = "_worker.js"`)
**ریموت:** `github.com/tivanmelhotra-creator/edgetunnel` (fork از `cmliu/edgetunnel`)
**شاخه فعال:** `main`

### معماری فعلی (`_worker.js`)
- **ورودی اصلی** (خط ۱۲ `export default { fetch }`) — مسیریابی کامل:
  - `/version` → اطلاعات نسخه
  - WebSocket upgrade → `处理WS请求` (پروکسی WS)
  - POST (gRPC/XHTTP) → `处理gRPC请求` / `处理XHTTP请求`
  - `/login`, `/admin`, `/admin/*` → پنل مدیریت (cookie auth)
  - `/sub` → تولید/تبدیل اشتراک (Clash/Sing-box/Surge/Base64)
  - `/locations`, `/robots.txt`
- **پروتکل‌ها:** VLESS (`解析魏烈思请求`)، Trojan (`解析木马请求`)، Shadowsocks (AEAD)
- **ترنسپورت:** WS، XHTTP، gRPC، WS Early-Data، GrainTCP (صف بالادست/پایین‌دست)
- **رمزنگاری دستی:** TLS client کامل (`TlsClient`)، ChaCha20-Poly1305، AES-GCM، HKDF، Poly1305
- **زنجیره پروکسی:** SOCKS5، HTTP(S)، TURN، SSTP(SoftEther)
- **اشتراک:** هات‌پچ Clash/Sing-box/Surge، تولیدکننده IP بهینه، DoH، CF Usage API
- **وابستگی فرانت‌اند:** صفحات استاتیک از `https://edt-pages.github.io`

### متغیرهای محیطی کلیدی
`ADMIN`(اجباری)، `KEY`، `UUID`، `PROXYIP`، `URL`، `GO2SOCKS5`، `DEBUG`، `OFF_LOG`، `BEST_SUB`، `PRELOAD_RACE_DIAL`

### نکات مهم
- کد کاملاً تک‌فایل و فشرده است (نام توابع فارسی/چینی) — بدون build step.
- ویرایش `_worker.js` مستقیم؛ تست با `wrangler dev` نیاز به KV binding دارد.
- `.github/workflows/` ممکن است permission محدود داشته باشد → قانون ویرایش دستی.

---

## 🗺️ استپ‌های توسعه

- [x] استپ ۱ — راه‌اندازی محیط توسعه و اعتبارسنجی پایه  ✅ 2026-06-06
  - [x] تأیید ابزار: node v20.20.2 / npm 10.8.2 / wrangler 4.86.0
  - [x] بررسی سینتکس `_worker.js` با `node --check` → SYNTAX_OK
  - [x] ساخت `package.json` (اسکریپت‌های check/dev/deploy، type=module)
  - [x] ساخت `wrangler.dev.toml` با KV binding محلی (فایل تولید دست‌نخورده)
  - [x] اجرای `wrangler dev --local` و تأیید `/version?uuid=...` → `{"Version":20260601154939}` HTTP 200
  - یادداشت: `http:` به `https:` ریدایرکت می‌شود (رفتار درست worker، خط ۷۴)

- [ ] استپ ۲ — مستندسازی نقشه کد و نقاط توسعه
  - استخراج جدول کامل مسیرها (routes) و توابع به `docs/ARCHITECTURE.md`
  - مستند کردن جریان داده WS/XHTTP/gRPC
  - مستند کردن جریان `/sub` و تبدیل اشتراک
  - فهرست متغیرهای محیطی و اثرشان
  - شناسایی و فهرست نقاط قابل بهبود (TODO list)

- [ ] استپ ۳ — بهبود پنل مدیریت و امنیت auth
  - بازبینی منطق cookie/`MD5MD5` و سخت‌سازی
  - افزودن اعتبارسنجی ورودی در endpointهای `admin/*` POST
  - بهبود پیام‌های خطای JSON یکدست
  - تست دستی login/logout/init
  - ثبت تغییرات در CHANGELOG

- [ ] استپ ۴ — بهبود سیستم اشتراک (`/sub`)
  - بازبینی هات‌پچ Clash/Sing-box/Surge
  - بهبود انتخاب IP بهینه و DoH fallback
  - تست خروجی برای کلاینت‌های مختلف (UA)
  - رفع موارد لبه‌ای شناسایی‌شده
  - ثبت در CHANGELOG

- [ ] استپ ۵ — بهینه‌سازی مسیر داغ ترنسپورت (WS/GrainTCP)
  - بازبینی صف بالادست/پایین‌دست و backpressure
  - تنظیم پارامترهای buffer/concurrency
  - بررسی نشت حافظه و بستن سوکت‌ها
  - تست عملکرد پایه
  - ثبت در CHANGELOG

- [ ] استپ ۶ — پایداری CI/CD و انتشار
  - بازبینی workflowهای GitHub (sync / auto-close PR)
  - افزودن lint/check workflow (در صورت permission → ویرایش دستی)
  - به‌روزرسانی README و wrangler.toml
  - تگ نسخه و آماده‌سازی release notes
  - تأیید نهایی استقرار

---

## 🐛 باگ‌های یافته‌شده
(فعلاً موردی ثبت نشده)
