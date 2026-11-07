# PRD — uTPro.Feature.AdProtect (MVP)

- **Version:** 0.1.0-draft (specification, chưa code)
- **Ngày:** 2026-11-01
- **Repo đích:** `D:\_GitHub\uTPro\uTPro.Feature.AdProtect.Src` (theo cặp thư mục `.Src` convention)
- **Status:** Draft — chờ duyệt trước khi scaffold

---

## 1. Tóm tắt & định vị

**AdProtect** là feature package cho Umbraco (16/17/18, net9.0/net10.0) bảo vệ tài
phí quảng cáo (đầu tiên là Google AdSense) trước **invalid traffic / invalid click**:
quyết định *per-visitor* có serve ad code hay không, đếm click-intent, tự cooldown
visitor click quá nhiều, và cung cấp backoffice quản lý.

**Tham chiếu đối thủ:** plugin WordPress *Ad Invalid Click Protector (AICP)*
(~100k installs, freemium $25–49). Trên **Umbraco Marketplace chưa có package tương
đương** — gap thật, niche nhỏ hơn WordPress nhưng chưa ai làm.

**Vai trò trong hệ sinh thái uTPro:** feature "mỏ neo" cho nhóm publisher/Content
site + dogfood ngay cho `foto.t4vn.com` (đặt ad đúng state) và T4VN (sau khi port
sang uTPro). Bối cảnh thực tế: T4VN dính *ad serving limit* ngày 2026-10-14 do bot
scanner + ad code dán trên mọi subdomain (kể cả trang lỗi) — đúng bài toán này.

## 2. Khách hàng & use case

| Đối tượng | Use case |
|---|---|
| Site Umbraco chạy AdSense | Chặn bot/scanner thấy quảng cáo; giới hạn click/IP/ngày |
| Site đa ngôn ngữ (EN/VN) | Tắt ad theo quốc gia không có demand quảng cáo |
| Chính T4VN/foto | Ad chỉ render ở state có nội dung public; số liệu click nội bộ |
| Agency xây site khách | Cấu hình ad an toàn mặc định, báo cáo cho khách |

## 3. Phạm vi

### 3.1 Goals (MVP)

1. Rule engine: Allow/Block theo **IP, CIDR, Country, ASN** — priority + enable + expire
2. Click-intent tracker (JS heuristic) + đếm per **IP** và per **anon-cookie**;
   vượt ngưỡng → cooldown tự động
3. `GET /api/adprotect/check` quyết định serve ad (cache-safe, per-request)
4. TagHelper `<utc-ad>` render slot + `<ins class="adsbyscript">` tuỳ chọn
5. Backoffice section: dashboard, rules CRUD, blocks list (unban), click log 7 ngày
6. Cấu hình 2 tầng: `appsettings.json` (mặc định khi cài) + **Settings UI trong
   backoffice** — sửa live, lưu DB, áp dụng ngay không restart
7. Tuân thủ AdSense ToS: chỉ *ẩn* ad, không sửa/đóng khung lôi kéo click

### 3.2 Non-goals (để v1.1+)

- Phát hiện ad-blocker visitor
- Thay thế quảng cáo bằng ad khác khi bị chặn (backfill)
- Per-site doctype config (MVP: global + per-host filter)
- Phân tích chi tiết như Ezoic (EPMV, layout testing)
- Tích hợp AdSense API kéo báo cáo doanh thu
- Anti-fingerprint nâng cao (device fingerprint, ML)

## 4. Kiến trúc

```
┌───────────────────────────── Umbraco site (net9/net10) ─────────────────────────┐
│                                                                                  │
│  Razor view                AdProtect middleware/endpoint        Backoffice       │
│  ┌───────────────┐  fetch  ┌──────────────────────┐   NPoco   ┌──────────────┐ │
│  │ <utc-ad>      │───────► │ GET /api/adprotect/  │──────────►│ Rule / Click │ │
│  │ (slot rỗng +  │         │       check          │  (3 bảng)  │ Block tables │ │
│  │  token HMAC)  │◄─────── │ POST /api/adprotect/ │           └──────┬───────┘ │
│  └──────┬────────┘  JSON   │       click          │                  │         │
│         │ inject          └──────────┬───────────┘          Section "AdProtect"│
│  AdProtect.js                        │ dùng service                          │
│  (mouseenter → blur/                 ▼                                       │
│   visibilitychange)        AdProtectService ← GeoLocationService (IP+country) │
│                            - Rules (MemoryCache 30s)                           │
│                            - Counters & cooldown                              │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Dependency: `uTPro.Feature.GeoLocation` (hard dependency)** — đã có sẵn: resolve
client IP đúng chuẩn proxy (`CF-Connecting-IP`, `X-Forwarded-For`), country qua
`CF-IPCountry` + MMDB offline. Không viết lại.

**Quyết định thiết kế #1 — Cache-safe:** quyết định serve/không-serve **không nằm
trong Razor** ( Razor render slot rỗng + token) mà nằm ở **fetch client-side tới
endpoint**. Nhờ vậy tương thích output cache và dùng được cho cả site tĩnh (Jekyll)
trỏ API về uTPro host.

**Quyết định thiết kế #2 — Click heuristic:** click vào iframe Google không đọc
được trực tiếp (cross-origin) → chuẩn ngành là: `mouseenter` vào container ad →
"arm"; `window blur` hoặc `visibilitychange→hidden` khi đang arm → tính là 1
click-intent. Đây là cách AICP dùng 10+ năm, **không vi phạm ToS AdSense**.

## 5. Schema DB (NPoco)

> Tên bảng theo convention các feature có DB (đối chiếu AuditLog khi scaffold).
> `Ip` lưu plain hoặc HMAC-SHA256 tuỳ `StoreHashedIPs` (mặc định false).

### 5.1 `uTProAdProtectRule`

| Cột | Kiểu | Ghi chú |
|---|---|---|
| Id | int PK identity | |
| Name | nvarchar(200) | tên rule |
| Type | nvarchar(16) | `Ip` \| `Cidr` \| `Country` \| `Asn` |
| Value | nvarchar(500) | IP đơn / CIDR / ISO-2 / số ASN |
| Mode | nvarchar(8) | `Block` \| `Allow` |
| Priority | int | nhỏ = đánh trước, mặc định 100 |
| Host | nvarchar(200) | rỗng = mọi site; `foto.t4vn.com` = chỉ host đó |
| Enabled | bit | |
| Note | nvarchar(500) | |
| ExpiresAt | datetime2 NULL | NULL = vĩnh viễn |
| CreatedAt / UpdatedAt | datetime2 | |

Index: `(Enabled, Type, Priority)`

### 5.2 `uTProAdProtectClickLog`

| Cột | Kiểu | Ghi chú |
|---|---|---|
| Id | bigint PK identity | |
| AnonId | nvarchar(64) | GUID cookie anon |
| Ip | nvarchar(64) | plain/hash |
| Country | char(2) NULL | |
| Host | nvarchar(200) | |
| Path | nvarchar(500) | |
| Slot | nvarchar(100) | |
| CreatedAt | datetime2 | |

Index: `(Ip, CreatedAt)`, `(AnonId, CreatedAt)`. Retention: purge > 30 ngày (background job như JobMonitor pattern, hoặc lazy purge trong endpoint).

### 5.3 `uTProAdProtectBlock` (cooldown runtime)

| Cột | Kiểu | Ghi chú |
|---|---|---|
| Id | int PK identity | |
| KeyType | nvarchar(8) | `Ip` \| `Cookie` |
| KeyValue | nvarchar(64) | |
| Reason | nvarchar(32) | `ClickBudget` \| `Manual` \| `RuleAuto` |
| BlockedUntil | datetime2 | |
| CreatedAt | datetime2 | |

Index: `(KeyType, KeyValue, BlockedUntil)`

### 5.4 `uTProAdProtectSetting` (settings chỉnh từ backoffice)

| Cột | Kiểu | Ghi chú |
|---|---|---|
| Id | int PK identity | |
| Host | nvarchar(200) | rỗng = global; `foto.t4vn.com` = override cho 1 site |
| MaxClicksPerIpPerDay | int | mặc định 10 |
| MaxClicksPerCookiePerDay | int | mặc định 3 |
| CooldownHours | int | mặc định 24 |
| StoreHashedIPs | bit | mặc định false |
| AdClientId | nvarchar(50) | publisher id cho TagHelper render `<ins>` |
| PlaceholderHtml | nvarchar(1000) | rỗng = collapse |
| UpdatedAt | datetime2 | |

- 1 dòng `Host = ''` (global) + 0..n dòng per-host. Read = MemoryCache 30s,
  invalidate khi save.
- `ApiPrefix` KHÔNG nằm ở đây (route phải biết trước khi DB sẵn sàng) — chỉ
  cấu hình được qua `appsettings.json`.

## 6. Rule engine — thứ tự đánh giá

Trả về quyết định **đầu tiên khớp** (nhỏ Priority trước; cùng priority → theo Id):

```
1. Block runtime (uTProAdProtectBlock còn hiệu lực)          → DENY (cooldown)
2. Rule Allow match                                          → ALLOW
3. Rule Block match (Ip → Cidr → Country → Asn)              → DENY
4. Click budget: IP ≥ MaxClicksPerIpPerDay HOẶC cookie ≥ …    → DENY (ghi cooldown)
5. Mặc định                                                  → ALLOW
```

- `Country` lấy từ GeoLocationService; `Asn` lookup offline từ TSV
  (iptoasn/cf `ASNum` — chọn 1 nguồn, ghi rõ ở Open questions).
- Rules cache trong `IMemoryCache` 30s; invalidate khi CRUD từ backoffice.
- Built-in seed: danh sách ASN datacenter (Amazon 16509/14618, Google 15169/396982,
  Microsoft 8075, DigitalOcean 14061, OVH 16276, Hetzner 24940, Vultr 20473,
  Choopa 45102, Tencent 132203) — **disabled mặc định**, bật 1 click.

## 7. API contract

Prefix mặc định `/api/adprotect` (configurable). Endpoint không đăng ký vào
`Umbraco.Cms` auth — public nhưng chống lạm dụng (mục 9).

### 7.1 `GET /check?slot={slot}`

```json
// 200 — luôn 200, kể cả deny (tránh attacker phân biệt trạng thái)
{ "serve": false }
```

- Header `Cache-Control: no-store` (quyết định per-visitor).
- Response không giải thích lý do ở public endpoint (lý do chỉ log server-side).

### 7.2 `POST /click`

Body: `{ "slot": "banner-top", "token": "<hmac>" }` → `{ "ok": true }`

- Token do TagHelper sinh: `HMAC(tokenKey, anonId|host|slot|exp10phút)` dùng
  `IDataProtector` (Data Protection của ASP.NET Core).
- Chấp nhận token ⇒ chống spam curl không qua trang; dedupe: 1 token tính 1 lần
  (cache token 15 phút).

### 7.3 Backoffice API (auth Umbraco backoffice)

`GET /dashboard?days=7`, `GET/POST/DELETE /rules`, `GET /blocks`, `POST /blocks/unban`,
`GET /clicks?days=7&ip=`.

## 8. Client-side — `adprotect.js` (~2–3KB, vanilla)

### 8.1 Gating flow

```
DOMContentLoaded
 → với mỗi [data-utpro-ad]:
   1. GET /api/adprotect/check?slot=…  (fetch, same-origin credentials)
   2. serve=true  → append <ins class="adsbybanner" …> đã render sẵn + push adsbygoogle
   3. serve=false → container.collapse (display:none) hoặc placeholder nếu
                    data-placeholder="true"
```

- Giả định: site đã load script adsbygoogle trong `<head>` (AICP cũng vậy).
  TagHelper có option render `<ins>` sẵn trong slot để JS chỉ `push({})`.
- Lưu ý docs: không dùng cho slot **Auto Ads** (auto inject mọi nơi, không kiểm soát
  per-slot được) — khuyến nghị unit thủ công.

### 8.2 Click-intent tracker

```
container mouseenter → armed = true; setTimeout(arm decay 3s)
window blur / visibilitychange→hidden khi armed:
   POST /click {slot, token}   (fire-and-forget, debounce ≥5s giữa 2 post)
   armed = false
```

- Không jQuery, CSP-friendly (không inline bắt buộc), respect `prefers-reduced-motion`
  không liên quan — nhưng phải hoạt động khi JS framework = none.

## 9. Bảo mật & chống lạm dụng

| Nguy cơ | Biện pháp |
|---|---|
| Spam `POST /click` để ban IP người khác | Chỉ dùng **IP server-resolved** (GeoLocation chain); body không chứa IP; HMAC token + dedupe |
| Đoán token | Data Protection key ring per-install, token hết hạn 10 phút |
| DoS 2 endpoint | In-memory token bucket ~60 req/phút/IP; Cloudflare WAF là tuyến 1 (docs khuyến nghị) |
| Storage IP (GDPR) | `StoreHashedIPs: true` → HMAC-SHA256(key, ip), vẫn so khớp được |
| CGNAT (Việt Nam — Viettel/VNPT share IP) | Budget đếm **cả cookie lẫn IP**; ngưỡng IP cao hơn (mặc định IP=10, cookie=3) |
| Lộ cấu hình | Endpoint public không trả reason/chi tiết rule |

**AdSense ToS note:** chỉ *ẩn* ad với một số visitor là hợp lệ (tiền lệ AICP);
tuyệt đối không sửa ad code, không tạo CTA "click ủng hộ" — ghi cảnh báo này vào docs.

## 10. Cấu hình — 2 tầng, thứ tự ưu tiên

**Backend đọc theo chuỗi:** DB Setting (Host match, rồi global) → appsettings.json → hard default.

### 10.1 `appsettings.json` (mặc định khi cài, admin không đụng tới thường xuyên)

```json
"AdProtect": {
  "ApiPrefix": "/api/adprotect",
  "SeedDatacenterAsnRule": false
}
```

### 10.2 Settings trong backoffice (sửa live, lưu bảng 5.4, không restart)

| Nhóm | Field | Kiểu |
|---|---|---|
| Click budget | MaxClicksPerIpPerDay / MaxClicksPerCookiePerDay / CooldownHours | int, min 1 |
| Quảng cáo | AdClientId / PlaceholderHtml | text |
| Riêng tư | StoreHashedIPs | bool (cảnh báo: bật/tắt làm mất so khớp IP cũ) |

Save → invalidate rules/settings cache → hiệu lực từ request tiếp theo (không restart, không deploy). UI có scope selector: **Global** hoặc chọn 1 host.

## 11. Backoffice section — "AdProtect"

| Màn | Nội dung |
|---|---|
| Dashboard | Card: serves today / blocks today / active cooldowns; Top 10 IP 7 ngày; chart nhỏ 14 ngày |
| Rules | Bảng CRUD + toggle Enabled + duplicate rule. Quick-add: nhập IP / CIDR / country / ASN nhanh ở thanh trên (block 1 IP = 3 click), bulk-import paste danh sách IP (mỗi dòng 1 giá trị) |
| Blocks | Danh sách hiệu lực + nút Unban |
| Click log | Filter theo IP/slot/khoảng ngày, phân trang |
| Settings | Form theo §10.2, scope Global/per-host, save là ăn ngay |

Section + menu item + icon theo pattern AuditLog/JobMonitor. Localization keys EN
(có thể thêm VI sau).

## 12. Testing

- **Unit:** CIDR matcher (v4/v6), rule priority order, token HMAC verify + dedupe,
  cooldown logic (IP vs cookie), hashed-IP match
- **Integration (TestSite):** check endpoint đúng header proxy; click endpoint từ
  chối token lạ/hết hạn; migration up/down NPoco
- **Thủ công (dogfood):** gắn lên `foto.t4vn.com` gallery — verify slot ẩn khi
  deny, click 4 lần trong ngày → cookie cooldown; đối chiếu Cloudflare Events
  (ASN challenge) và AdSense pageviews giảm ở traffic cloud

## 13. Packaging checklist (theo 12 feature hiện có)

- [ ] `uTPro.Feature.AdProtect.Src/` — sln, `src/uTPro.Feature.AdProtect`,
      `src/uTPro.Feature.AdProtect.TestSite`
- [ ] Multi-target `net9.0`/`net10.0`; Umbraco 16/17/18 dependency range
- [ ] Migration qua `builder.PackageMigrationPlans()` (KHÔNG phải `.Migrations()`)
- [ ] NPoco tables (mục 5) — không EF Core
- [ ] `pack.ps1`, `umbraco-marketplace.json`, `icon.png`, `LICENSE.txt`,
      `Image/Screenshots/`, `docs/` (getting-started, configuration, integration-utpro)
- [ ] DependsOn: `uTPro.Feature.GeoLocation` (NuGet)
- [ ] README bản `.Src` mirror từ bản docs folder

## 14. Milestones

| Mốc | Nội dung | Ước lượng |
|---|---|---|
| M1 | Schema + rule engine + 2 endpoint + TagHelper + JS (không UI) | 2–3 ngày |
| M2 | Backoffice section (dashboard/rules/blocks/log/settings) + localization | 1–2 ngày |
| M3 | Pack, docs, TestSite dogfood foto.t4vn.com | 0.5–1 ngày |

## 15. Cây thư mục dự kiến

```
uTPro.Feature.AdProtect/            # (đã có) docs home
├── README.md                       # ✅ xong
├── PRD-MVP.md                      # ✅ file này
├── docs/                           # viết ở M3
├── Image/Screenshots/              # chụp ở M3
├── icon.png
├── LICENSE.txt
└── umbraco-marketplace.json

uTPro.Feature.AdProtect.Src/
├── AGENTS.md
├── Build/
├── pack.ps1
├── umbraco-marketplace.json
├── uTPro.Feature.AdProtect.sln
└── src/
    ├── uTPro.Feature.AdProtect/
    │   ├── AdProtectService.cs          # rule engine + counters
    │   ├── AdProtectEndpoints.cs        # map endpoint check/click
    │   ├── AdProtectTagHelper.cs        # <utc-ad>
    │   ├── Tokens/AdProtectTokenService.cs
    │   ├── Models/                      # POCO NPoco + rule DTOs
    │   ├── Migrations/
    │   ├── Section/                     # section, dashboard, API controllers
    │   └── wwwroot/js/adprotect.js
    └── uTPro.Feature.AdProtect.TestSite/
```

## 16. Open questions (chờ quyết trước M1)

1. **Nguồn ASN lookup offline:** TSV iptuasn tự update, hay cache bảng ASN Cloudflare?
   (MVP: seed tĩnh trong bảng Rule + docs hướng dẫn thêm tay)
2. **Tên bảng** — đối chiếu convention đặt tên của AuditLog khi scaffold.
3. **Endpoint route:** giữ `/api/adprotect` riêng hay đăng ký under
   `/umbraco/api/`? (Khuyến nghị riêng — site tĩnh gọi được)
4. **License tier:** Free hoàn toàn như GeoLocation, hay Free + Pro (multi-site
   per-doctype config, scheduled reports) giống mô hình AICP?
5. **Tên hiển thị marketplace:** "AdProtect" có trùng nhãn hiệu đáng kể không —
   kiểm tra nhanh trước khi chốt slug `utpro.feature.adprotect`.
