# Flight Search - OpenClaw Skill

An OpenClaw skill that compares flight prices across airlines using the browser tool. Uses smart URL construction to skip complex UI interactions entirely.

## How It Works

Instead of trying to fill in search forms field-by-field (which breaks on JavaScript-heavy sites like Google Flights), this skill constructs URLs with search parameters pre-filled and navigates directly to results.

### Supported Sources

| Source | Status | Method |
|--------|--------|--------|
| Google Flights | Working | `q=` natural language parameter |
| Trip.com | Working | URL path + query parameters |
| Kayak | Blocked | Bot detection |
| Skyscanner | Blocked | CAPTCHA |
| Ctrip (携程) | Blocked | Requires login |

### Strategy Chain

The skill follows a strict fallback chain — each strategy gets one attempt, no retry loops:

1. **Google Flights** (primary) - pre-filled URL, efficient snapshot extraction
2. **Trip.com** (complement/fallback) - URL params + JavaScript `innerText` extraction
3. **Individual airline websites** - direct navigation with form fill
4. **Web search** - last resort, estimates only

## Output Example

```
✈️ 上海 → 温哥华 | 2026-02-25 | 商务舱

—— 直飞 ——

1️⃣ 加拿大航空 Air Canada | $7,521 / ¥54,602 / C$10,530
   PVG 17:35 ✈ YVR 11:50 | 10h15m 直飞

—— 转机 ——

2️⃣ 国泰航空 Cathay Pacific | $2,164 / ¥15,711 / C$3,030
   总时长 17h30m
   第1段: PVG 9:30 ✈ HKG 12:20 (2h50m)
   ⏳ 香港HKG转机 等待2h50m
   第2段: HKG 15:10 ✈ YVR 11:00 (11h50m)

3️⃣ 大韩航空 Korean Air | $3,724 / ¥27,039 / C$5,214
   总时长 13h15m
   第1段: PVG 14:00 ✈ ICN 17:20 (2h20m)
   ⏳ 首尔ICN转机 等待1h20m
   第2段: ICN 18:40 ✈ YVR 11:15 (9h35m)

数据来源: Google Flights | 查询时间: 2026-02-25 18:30 UTC
汇率: 1 USD = ¥7.26 CNY = C$1.40 CAD (实时)
⚠️ 价格为实时查询结果，可能随时变动，以实际预订为准。
```

### Key Features

- **URL-first approach** — no fragile UI interactions on Google Flights
- **Per-leg breakdown** for connecting flights — departure/arrival times, flight duration, layover city and wait time for each segment
- **Tri-currency pricing** — USD, RMB (CNY), and CAD with live exchange rates
- **Overnight layover warnings** — flagged with ⚠️过夜
- **Anti-loop protection** — strict one-attempt-per-strategy rule prevents the common failure mode of AI getting stuck retrying the same broken approach

## URL Patterns

### Google Flights

```
https://www.google.com/travel/flights?q=one+way+business+class+flights+from+Shanghai+to+Vancouver+on+Feb+25+2026
```

The `q=` parameter accepts natural language and pre-fills trip type, cabin class, route, and date.

### Trip.com

```
https://www.trip.com/flights/shanghai-to-vancouver/tickets-sha-yvr?dcity=sha&acity=yvr&ddate=2026-02-25&flighttype=ow&class=c&lowpricecalendar=close&adult=1
```

Parameters: `flighttype` (ow/rt), `class` (y/c/f), `ddate` (YYYY-MM-DD).

## Requirements

- OpenClaw with browser tool enabled
- Chromium/Chrome installed and configured in `~/.openclaw/openclaw.json`

## Installation

Copy the `SKILL.md` file to your OpenClaw skills directory:

```bash
mkdir -p ~/.openclaw/workspace/skills/flight-search
cp SKILL.md ~/.openclaw/workspace/skills/flight-search/
```

Or clone this repo directly:

```bash
git clone https://github.com/chengli1986/flight-search.git ~/.openclaw/workspace/skills/flight-search
```
