# معماری edgetunnel — نقشه کد

> نسخه کد: `2026-06-01 15:49:39` — فایل: `_worker.js` (~۵۹۰۵ خط، تک‌فایل، بدون build)

## ۱. ساختار کلی
کل برنامه یک Cloudflare Worker تک‌فایل است با یک ورودی `export default { async fetch(request, env, ctx) }` (خط ۱۲). توابع با نام‌های فارسی/چینی نوشته شده‌اند. هیچ ماژول خارجی import نمی‌شود؛ همه رمزنگاری‌ها (TLS/AES-GCM/ChaCha20/Poly1305/HKDF) دستی پیاده شده‌اند.

```
fetch()
 ├─ نرمال‌سازی URL (حذف \ و %5C، تبدیل %3f→?)
 ├─ محاسبه userID/uuid از MD5MD5(ADMIN + KEY)
 ├─ تعیین 反代IP (PROXYIP env یا colo.PrOxYIp.CmLiUsSsS.net)
 ├─ تنظیم SOCKS5白名单 (GO2SOCKS5)
 └─ مسیریابی (جدول بخش ۲)
```

## ۲. جدول مسیرها (Routes)

| مسیر / شرط | متد | هندلر | توضیح |
| :--- | :---: | :--- | :--- |
| `/version?uuid=<userID>` | GET | inline | خروجی JSON نسخه |
| `Upgrade: websocket` | — | `处理WS请求` | پروکسی WebSocket (VLESS/Trojan/SS) |
| POST + `content-type: application/grpc` | POST | `处理gRPC请求` | پروکسی gRPC |
| POST (غیر admin/login) | POST | `处理XHTTP请求` | پروکسی XHTTP |
| `/<KEY>` | GET | inline | اشتراک سریع → ریدایرکت به `/sub` |
| `/login` | GET/POST | inline + Pages静态 | صفحه و احراز هویت ورود |
| `/admin`, `/admin/*` | GET/POST | inline | پنل مدیریت (cookie auth) |
| `/admin/log.json` | GET | KV `log.json` | لاگ‌ها |
| `/admin/getCloudflareUsage` | GET | `getCloudflareUsage` | آمار مصرف CF |
| `/admin/getADDAPI` | GET | `请求优选API` | اعتبارسنجی API بهینه |
| `/admin/check` | GET | `socks5/turn/sstp/httpConnect` + `TlsClient` | تست سلامت پروکسی |
| `/admin/init` | GET | `读取config_JSON(...,true)` | بازنشانی پیکربندی |
| `/admin/config.json` | GET/POST | KV `config.json` | خواندن/ذخیره پیکربندی |
| `/admin/cf.json` | GET/POST | KV `cf.json` | اعتبارنامه CF API |
| `/admin/tg.json` | POST | KV `tg.json` | پیکربندی تلگرام |
| `/admin/ADD.txt` | GET/POST | KV `ADD.txt` | IPهای بهینه دستی |
| `/logout` یا مسیر UUID | GET | inline | پاک کردن cookie |
| `/sub` | GET | inline (بخش ۴) | تولید/تبدیل اشتراک |
| `/locations` | GET | inline | فهرست locations رِله |
| `/robots.txt` | GET | inline | Disallow همه |
| پیش‌فرض | — | `nginx()` / `html1101()` / Pages | صفحه استتار |

## ۳. احراز هویت
- **رمز:** `env.ADMIN` (یا fallbackها: admin/PASSWORD/TOKEN/KEY/UUID).
- **userID/uuid:** `MD5MD5(ADMIN + KEY)` فرمت‌شده به UUIDv4 — مگر `env.UUID` معتبر باشد (override).
- **Cookie ادمین:** `auth = MD5MD5(UA + KEY + ADMIN)`، `HttpOnly; Secure; SameSite=Strict; Max-Age=86400`.
- **TOKEN اشتراک:** `MD5MD5(host + userID)`.
- **TOKEN تبدیل اشتراک (روزانه):** `MD5MD5(base64SecretEncode(订阅TOKEN, userID) + 日序号)` — امروز/دیروز معتبرند.

## ۴. جریان اشتراک `/sub`
1. اعتبارسنجی توکن (کاربر / بک‌اند تبدیل / تولیدکننده بهینه).
2. خواندن `config_JSON` از KV (`读取config_JSON`).
3. تشخیص نوع خروجی از UA/پارامتر: `clash` / `singbox` / `surge` / `quanx` / `loon` / `mixed`.
4. تولید نودها:
   - **local:** IP بهینه از `生成随机IP` یا `ADD.txt`؛ APIها از `请求优选API`.
   - **生成器:** از `获取优选订阅生成器数据`.
5. ساخت لینک VLESS/Trojan/SS با مسیر، ECH، TLS分片، 链式代理(زنجیره).
6. هات‌پچ بر اساس نوع: `Clash订阅配置文件热补丁` / `Singbox...热补丁` / `Surge...热补丁`.
7. جایگزینی placeholder UUID/example.com با مقادیر واقعی + خروجی Base64 در صورت نیاز.

## ۵. پروتکل‌ها و ترنسپورت

### پارسرهای پروتکل
- `解析魏烈思请求` — VLESS (UUID ۱۶ بایتی، subarray کم‌کپی)
- `解析木马请求` — Trojan (SHA-224 رمز)
- Shadowsocks AEAD — `SS派生主密钥/会话密钥`, `SSAEAD加密/解密`

### ترنسپورت‌ها
- `处理WS请求` — WebSocket + Early-Data (`解码WS早期数据`، سقف ۸KB)
- `处理XHTTP请求` — XHTTP (`读取XHTTP首包`)
- `处理gRPC请求` — gRPC
- **GrainTCP:** `创建上行写入队列` (سقف 16MB/4096 آیتم،合包 16KB) + `创建下行Grain发送器` (پکت 32KB)

### فوروارد و زنجیره پروکسی
- `forwardataTCP` — اتصال TCP با تا ۲ مسیر هم‌زمان (`TCP并发拨号数`؛ CMCC=۱)
- `forwardataudp` / `转发木马UDP数据` — UDP over TCP (DNS)
- `socks5Connect` / `httpConnect` / `httpsConnect` — زنجیره SOCKS5/HTTP(S)
- `turnConnect` (STUN/TURN) / `sstpConnect` (SoftEther)
- `创建请求TCP连接器` — انتخاب کانکتور بر اساس request

### پشته TLS دستی
`TlsClient` (خط ۳۰۳۸) + `buildClientHello`/`parseServerHello`/`TlsRecordParser`/`TlsHandshakeParser` + اولیه‌های `chacha20Poly1305*`, `aesGcm*`, `hkdfExpandLabel`, `tls12Prf`.

## ۶. توابع کمکی کلیدی
| تابع | کار |
| :--- | :--- |
| `读取config_JSON` | بارگذاری/ساخت پیکربندی از KV |
| `MD5MD5` | هش دولایه MD5 (مبنای توکن‌ها) |
| `DoH查询` | حل DNS از طریق DoH (TXT→A→AAAA) |
| `生成随机IP` / `请求优选API` | منابع IP بهینه |
| `识别运营商` | تشخیص اپراتور (cmcc و...) |
| `getCloudflareUsage` | آمار مصرف Workers/Pages |
| `解析地址端口` / `获取SOCKS5账号` | پارس پروکسی |
| `nginx()` / `html1101()` | صفحات استتار |

## ۷. متغیرهای محیطی
| متغیر | اجباری | اثر |
| :--- | :---: | :--- |
| `ADMIN` | ✅ | رمز پنل + مبنای userID |
| `KEY` | ❌ | کلید رمز توکن + مسیر اشتراک سریع |
| `UUID` | ❌ | override userID (باید UUIDv4) |
| `PROXYIP` | ❌ | رِله سراسری (خاموش‌کردن兜底) |
| `URL` | ❌ | صفحه استتار |
| `GO2SOCKS5` | ❌ | لیست اجبار SOCKS5 (append) |
| `DEBUG` | ❌ | لاگ console |
| `OFF_LOG` | ❌ | خاموش‌کردن لاگ KV |
| `BEST_SUB` | ❌ | حالت تولیدکننده اشتراک بهینه |
| `PRELOAD_RACE_DIAL` | ❌ | پیش‌بارگذاری رقابتی TCP |

## ۸. ذخیره‌سازی (KV)
`config.json`, `cf.json`, `tg.json`, `ADD.txt`, `log.json` — همگی در binding `KV`.

## ۹. وابستگی فرانت‌اند
صفحات استاتیک پنل از `https://edt-pages.github.io` (`Pages静态页面`) fetch می‌شوند؛ سورس جدا: `EDT-Pages/EDT-Pages.github.io`.
