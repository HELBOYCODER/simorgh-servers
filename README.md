# Simorgh Servers

Automatic public-server feeds for **Simorgh**, a macOS client. This
repository holds the seed feeds, the Telegram channel list, and the CI that
harvests, tests, signs, and publishes working server configurations. It is a
rebranded port of the server-discovery mechanism of
[zeghostwriter/ZeroNet](https://github.com/zeghostwriter/ZeroNet) (MIT; see
[NOTICE](NOTICE)).

## What this repository publishes

Three scheduled workflows (`harvest` every 2 h, `fresh` every 10 min, `crowd`
every 20 min) fetch the public feeds in
[`deploy/crowd/sources.json`](deploy/crowd/sources.json) and
[`deploy/crowd/harvest-sources.json`](deploy/crowd/harvest-sources.json), plus
the configs posted in the Telegram channels of
[`deploy/crowd/telegram-channels.json`](deploy/crowd/telegram-channels.json).
Every config is tested the way the app tests it (TCP, then a real request with
TLS confirmed); only working, encrypted ones are kept. Results are published
as a **single-commit `crowd-data` branch**:

| File on `crowd-data` | What it is |
| --- | --- |
| `verified.txt` | Working tested server links, one per line, best first. |
| `verified.txt.sig` | Detached signature of `verified.txt`: one line `ed25519:<hex>`. |
| `rankings.json` | Which public servers and clean Cloudflare addresses work on which network, built from crowd reports. |
| `rankings.json.sig` | Detached signature of `rankings.json`, same format. |
| `fresh.txt`, `fresh-state.json` | The configs the 10-minute `fresh` workflow keeps at the top, and its read position. |
| `telegram-state.json` | Telegram channels found by earlier harvest runs and where each was last read. |
| `modes/` | Per-day connection-mode totals, sealed so only the maintainer can read them (only when `MODE_STATS_KEY` is set). |

The tools that do this work (`simorgh-harvest`, `simorgh-crowd`,
`simorgh-sign`, `simorgh-stats`) live in
[HELBOYCODER/simorgh-core](https://github.com/HELBOYCODER/simorgh-core). The
workflows clone it at `SIMORGH_CORE_REF` — currently the placeholder `main`;
it must be pinned to a release tag once simorgh-core publishes one. The tool
names are placeholders for the rebranded binaries; simorgh-core still ships
the upstream names (`zero-discovery` with `zeronet-*` bins) today, so the
workflows must be flipped together with that rebrand.

## URLs the Simorgh macOS app reads

In order tried, mirrors first after GitHub raw:

```
https://raw.githubusercontent.com/HELBOYCODER/simorgh-servers/crowd-data/verified.txt
https://cdn.jsdelivr.net/gh/HELBOYCODER/simorgh-servers@crowd-data/verified.txt
https://fastly.jsdelivr.net/gh/HELBOYCODER/simorgh-servers@crowd-data/verified.txt
```

`rankings.json` at the same three locations, each with a `.sig` sibling. The
`verified-source.json` on the `main` branch describes the same list in feed
form, so other tools can add it as an ordinary feed:

```
https://raw.githubusercontent.com/HELBOYCODER/simorgh-servers/main/deploy/crowd/verified-source.json
```

The companion `FEEDS-NOTES.md` in the Simorgh workspace documents the exact
client-side behavior (cache TTL, replay protection, signature enforcement).

## How to add a feed

Open a pull request that adds one entry to
[`deploy/crowd/sources.json`](deploy/crowd/sources.json):

```json
{ "id": "your-feed-id", "url": "https://raw.githubusercontent.com/…/all.txt", "tier": 2 }
```

- `url` must serve plain text: one server link per line (`vless://`,
  `vmess://`, `trojan://`, `ss://`, `hysteria2://`, …) or base64/b64url
  blobs of the same, with `#comment` lines allowed.
- `tier` is fetch priority: 1 = fetched first.
- Feeds are **not** trusted blindly: every config is tested before it reaches
  `verified.txt`, and only encrypted transports are kept.
The `crowd` workflow re-reads this file when ranking, so feeds added here also
widen what crowd reports can name.

## How signing works

`verified.txt` and `rankings.json` can introduce servers the app has never
seen, and they are fetched over censored links where a CDN or a filtering
network could serve a substitute. So CI signs each file with an **Ed25519**
key pair:

1. A maintainer runs `simorgh-sign keygen` once. The **public key** hex is
   pasted into simorgh-core (`sign::PUBLIC_KEY_HEX`) and compiled into the
   app; the **private seed** is stored in this repository's settings as the
   secret `CROWD_SIGNING_KEY` (Settings → Secrets and variables → Actions).
2. Every workflow run signs the published file into `<name>.sig` (one line,
   `ed25519:<hex>`) and re-verifies it before pushing; a wrong secret stops
   the run rather than publishing lists nobody accepts.
3. The app fetches `<url>.sig` next to the file and refuses any body that
   does not verify against the compiled-in key. While no key is configured in
   a build, and while this repository has no `CROWD_SIGNING_KEY` secret, the
   workflows publish unsigned and the app falls back to trusting the source
   (the `harvest`/`crowd` logs warn about this).

The private key never enters this repository. Signatures prove *who*
published a file; freshness comes from `generated_at` monotonicity in the
app, which refuses replayed older files.

## Required repository configuration (GitHub)

- Secret `CROWD_SIGNING_KEY` — Ed25519 seed (base64 or hex), optional until a
  key pair exists.
- Variable `CROWD_RELAYS` — comma-separated crowd-relay URLs, or the
  `CLOUDFLARE_API_TOKEN` / `CLOUDFLARE_ACCOUNT_ID` secrets plus a
  `deploy/crowd-relay/wrangler.toml` for automatic discovery (as upstream
  does). Without any relay the `crowd` workflow publishes nothing, same as
  upstream.
- Secret `CROWD_EXPORT_TOKEN` — bearer token the `crowd` workflow uses to
  export reports from the relays.
- Variable `MODE_STATS_KEY` (optional) — public key for the sealed mode
  totals.
- The `crowd-data` branch is created automatically by the first publish; it
  is never merged into `main`.

---

## بخش فارسی (فارسی)

<div dir="rtl">

### این مخزن چیست؟

این مخزن فهرست سرویس‌های عمومی سالم را برای اپ macOS **سیمورغ** به‌طور خودکار
آماده می‌کند. سه گردش کاری زمان‌بندی‌شده (هر ۲ ساعت، هر ۱۰ دقیقه و هر ۲۰ دقیقه)
فیدهای عمومی موجود در `deploy/crowd/sources.json` و کانال‌های تلگرامی موجود در
`deploy/crowd/telegram-channels.json` را می‌خوانند، هر کانفیگ را همان‌طور که اپ
تست می‌کند می‌آزمایند، و فقط کانفیگ‌های سالم و رمزنگاری‌شده را در شاخه‌ی
`crowd-data` منتشر می‌کنند: `verified.txt` (سرورهای تست‌شده)، `rankings.json`
(کدام سرور روی کدام شبکه کار می‌کند)، و امضای هر فایل با پسوند `.sig`.

### آدرس‌هایی که اپ سیمورغ می‌خواند؟

۱. `verified.txt` و ۲. `rankings.json` از نشانی‌های زیر، به همین ترتیب:

- `https://raw.githubusercontent.com/HELBOYCODER/simorgh-servers/crowd-data/…`
- `https://cdn.jsdelivr.net/gh/HELBOYCODER/simorgh-servers@crowd-data/…`
- `https://fastly.jsdelivr.net/gh/HELBOYCODER/simorgh-servers@crowd-data/…`

هر فایل امضای جداگانه‌ای در `<name>.sig` دارد. فهرست کانال‌های تلگرامی و
فیدهای این مخزن از شاخه‌ی `main` خوانده می‌شوند.

### چطور یک فید اضافه کنیم؟

یک PR بزنید و یک ورودی به `deploy/crowd/sources.json` اضافه کنید: نشانی
متن‌خالصی که هر خط آن یک لینک سرور (`vless://`، `vmess://`، `trojan://`،
`ss://`، `hysteria2://`) است، همراه با `id` یکتا و `tier` (اولویت ۱ تا ۳).
هیچ فیدی کورکورانه قابل اعتماد نیست: همه‌ی کانفیگ‌ها قبل از انتشار آزمایش
می‌شوند و فقط ترافیک رمزنگاری‌شده نگه داشته می‌شود.

### امضا چگونه کار می‌کند؟

از آنجا که این فهرست‌ها می‌توانند سرورهای تازه را به اپ معرفی کنند و روی
شبکه‌های فیلترکننده دانلود می‌شوند، CI هر فایل را با کلید Ed25519 امضا می‌کند.
نیمه‌ی عمومی کلید در خود اپ (در simorgh-core) کامپایل شده و نیمه‌ی خصوصی فقط
به‌عنوان راز `CROWD_SIGNING_KEY` در تنظیمات همین مخزن نگه داشته می‌شود و هرگز
وارد مخزن نمی‌شود. فایل امضا یک خط به شکل `ed25519:<hex>` در کنار فایل اصلی
منتشر می‌شود و اپ هر فایلی را که امضای آن با کلید داخلی راستی‌آزمایی نشود رد
می‌کند. اگر کلیدی ساخته نشده باشد، فهرست‌ها بدون امضا منتشر می‌شوند و اپ تا
زمان تنظیم کلید، فهرست را قابل اعتماد می‌شمارد.

### اعتبار

مکانیزم کشف سرور در این مخزن برداشت بازنشانی‌شده از پروژه‌ی
[zeghostwriter/ZeroNet](https://github.com/zeghostwriter/ZeroNet) با مجوز MIT
است (بخش‌نامه‌ی `NOTICE` را ببینید). فیدهای شخص ثالث فهرست‌شده متعلق به
نگه‌دارندگان خودشان است و این پروژه آن‌ها را تأیید نمی‌کند. هیچ سروری در این
مخزن توسط پروژه‌ی سیمورغ بهره‌برداری نمی‌شود.

</div>
