---
name: Flight Search
slug: flight-search
description: Compare flight prices across airlines using browser tool. Smart URL construction avoids complex UI interactions. Google Flights primary, airline websites fallback.
metadata: {"openclaw":{"emoji":"✈️","requires":{"bins":[]},"os":["linux","darwin","win32"]}}
---

# Flight Search & Price Comparison

## When to Use

User wants to search, compare, or book flights using the browser tool.

## CRITICAL RULES

### Rule 1: NEVER retry a failing UI approach
If a browser action fails, **do NOT** apologize and retry the same approach. Switch to the next strategy immediately. No loops.

### Rule 2: URL-first, UI-last
Always construct URLs with search parameters pre-filled. **NEVER** fill in search forms field-by-field on Google Flights or similar complex SPAs. The `q=` parameter does all the work.

### Rule 3: Snapshot before every action
Always take a snapshot before clicking/typing. Refs go stale after any navigation or dynamic update.

### Rule 4: One strategy, one attempt
Each strategy gets ONE attempt. If it fails, move to the next. Do not loop.

### Rule 5: Wait after navigation
Flight results load dynamically. ALWAYS wait 6-8 seconds after navigation before taking a snapshot.

---

## Strategy Chain (follow in order)

### Strategy 1: Google Flights via `q=` Parameter (PRIMARY - TESTED & WORKING)

Google Flights accepts a natural language `q=` parameter that pre-fills all search criteria. **No UI interaction needed.**

**URL Template:**
```
https://www.google.com/travel/flights?q={TRIP_TYPE}+{CABIN}+class+flights+from+{ORIGIN_CITY}+to+{DEST_CITY}+on+{DATE_HUMAN}
```

**Examples:**
```
# One-way, business class, Shanghai to Vancouver
https://www.google.com/travel/flights?q=one+way+business+class+flights+from+Shanghai+to+Vancouver+on+Feb+25+2026

# Round trip, economy
https://www.google.com/travel/flights?q=round+trip+economy+flights+from+Beijing+to+Tokyo+on+Mar+15+2026+return+Mar+22+2026

# One-way, first class
https://www.google.com/travel/flights?q=one+way+first+class+flights+from+Guangzhou+to+Los+Angeles+on+Apr+10+2026
```

**After navigating:**
1. Wait 6 seconds: `browser act` with `kind: "wait"`, `timeMs: 6000`
2. Take efficient snapshot: `browser snapshot` with `mode: "efficient"`
3. Extract flight data from link elements — each flight appears as a link with full details:
   - Price (e.g., "From 2164 US dollars")
   - Airline name
   - Departure/arrival times and airports
   - Duration and number of stops
   - Layover details

**Filtering by specific airlines:** If the user wants only specific airlines (e.g., Air Canada, Cathay Pacific, China Eastern), extract ALL results from the snapshot, then filter the output table to show only the requested airlines. Google Flights shows all available flights by default.

**Currency:** Results default to USD. To change currency, click the "Currency" button at the bottom of the page after results load. Or append `&curr=CNY` to the URL (may not always work — use the button if needed).

### Strategy 2: Trip.com (TESTED & WORKING - good complement to Google Flights)

Trip.com (携程国际版) works with headless browsers and shows different pricing than Google Flights. Use it for price comparison or as fallback.

**URL Template:**
```
https://www.trip.com/flights/{ORIGIN_CITY_LOWER}-to-{DEST_CITY_LOWER}/tickets-{ORIGIN_LOWER}-{DEST_LOWER}?dcity={ORIGIN_LOWER}&acity={DEST_LOWER}&ddate={DATE}&flighttype={TRIP_TYPE}&class={CABIN}&lowpricecalendar=close&adult=1
```

**Parameters:**
- `{ORIGIN_CITY_LOWER}`: lowercase city name (e.g., `shanghai`)
- `{ORIGIN_LOWER}`: lowercase IATA code (e.g., `sha` for all Shanghai airports, or `pvg` for Pudong only)
- `{DATE}`: YYYY-MM-DD format
- `{TRIP_TYPE}`: `ow` = one-way, `rt` = round trip
- `{CABIN}`: `y` = economy, `c` = business, `f` = first

**Examples:**
```
# One-way, business class, Shanghai to Vancouver
https://www.trip.com/flights/shanghai-to-vancouver/tickets-sha-yvr?dcity=sha&acity=yvr&ddate=2026-02-25&flighttype=ow&class=c&lowpricecalendar=close&adult=1

# Round trip, economy, Beijing to Tokyo
https://www.trip.com/flights/beijing-to-tokyo/tickets-bjs-tyo?dcity=bjs&acity=tyo&ddate=2026-03-15&rdate=2026-03-22&flighttype=rt&class=y&lowpricecalendar=close&adult=1
```

**After navigating:**
1. Wait 8 seconds: `browser act` with `kind: "wait"`, `timeMs: 8000`
2. **Best method — JavaScript extraction** (most reliable on Trip.com):
   ```javascript
   browser evaluate --fn "JSON.stringify(Array.from(document.querySelectorAll('[data-testid^=\"u-flight-card\"]')).slice(0,15).map(c => c.innerText))"
   ```
   Each card returns text like: `"Air Canada | 17:35 | PVG T2 | 10h 15m | Nonstop | 11:50 | YVR M | US$7,606"`
3. **Alternative — efficient snapshot**: `browser snapshot` with `mode: "efficient"` — shows interactive elements but less flight detail text
4. **Alternative — screenshot**: `browser screenshot --full-page` — visual confirmation of results

**Note:** Trip.com may show a notification popup ("Allow notifications"). Dismiss it by clicking the X or just ignore it — it doesn't block the results.

**Note on Ctrip (ctrip.com):** The Chinese domestic version requires login and may not show results. Always use Trip.com (trip.com) instead.

### Strategy 3: Individual Airline Websites

Go directly to each airline's website. Most airline sites require form interaction, so use snapshot + fill approach carefully.

**Air Canada:** `https://www.aircanada.com/`
**China Eastern (东方航空):** `https://www.ceair.com/` or `https://us.ceair.com/en/`
**Cathay Pacific (国泰航空):** `https://www.cathaypacific.com/`
**Hainan Airlines (海南航空):** `https://www.hainanairlines.com/`
**Korean Air (大韩航空):** `https://www.koreanair.com/`
**Singapore Airlines:** `https://www.singaporeair.com/`
**ANA (全日空):** `https://www.ana.co.jp/en/`
**China Airlines (中华航空):** `https://www.china-airlines.com/`
**EVA Air (长荣航空):** `https://www.evaair.com/`

For each airline:
1. Navigate to the URL
2. Wait for load (6-8 seconds)
3. Take efficient snapshot
4. If search form present, use `fill` kind to set all fields at once (NOT one by one)
5. If `fill` fails or is not applicable, try `type` + `click` on the search button
6. If ANY action fails, skip this airline and move to the next

### Strategy 4: Web Search Fallback

If all browser strategies fail, use web search:
```
web_search: "{AIRLINE} {ORIGIN} to {DEST} {DATE} {CABIN} class ticket price"
```

Clearly tell the user that real-time browser scraping failed and these are search-based estimates only.

---

## BLOCKED SITES (Do NOT attempt)

These sites detect and block headless browsers. Skip them entirely:

| Site | Block Method |
|------|-------------|
| **Kayak** | Bot detection page ("What is a bot?") |
| **Skyscanner** | CAPTCHA ("Are you a person or a robot?") |
| **Expedia** | Bot detection |

Do NOT waste time on these sites. Go directly to Google Flights.

---

## Browser Best Practices for Flight Sites

### Snapshot Strategy
- Use `mode: "efficient"` — flight pages are large, efficient mode reduces noise
- Each flight result in Google Flights appears as a `link` element with ALL flight info in the text
- Parse the link text to extract: price, airline, times, airports, duration, stops

### Wait Strategy
```json
{"action": "act", "request": {"kind": "wait", "timeMs": 6000}}
```
- ALWAYS wait 6-8 seconds after navigating to a flight site
- If snapshot shows loading state or empty results, wait 5 more seconds and retry snapshot only (NOT re-navigate)

### Error Recovery
| Error | Action |
|-------|--------|
| "element not found" | Take fresh snapshot, check if page changed |
| "strict mode violation" | Element matches multiple — use more specific ref |
| "navigation timeout" | Site may be blocking — move to next strategy |
| "element covered" | Close any popups/overlays first, then retry |
| Bot detection page | SKIP to next strategy immediately |
| Same error twice | STOP. Move to next strategy. |

### Anti-Bot Detection Signs
- CAPTCHA or "Press & Hold" challenge
- "Access Denied" or 403 error
- "What is a bot?" page
- Cloudflare challenge page
- Blank page with no content

**If detected: skip immediately. Do NOT try to solve CAPTCHAs or bypass bot detection.**

---

## Output Format

Present results as a comparison table in the user's language:

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

4️⃣ 厦门航空 XiamenAir | $1,734 / ¥12,589 / C$2,428
   总时长 35h40m
   第1段: SHA 22:00 ✈ XMN 00:15+1 (2h15m)
   ⏳ 厦门XMN转机 等待22h45m ⚠️过夜
   第2段: XMN 23:00 ✈ YVR 17:40 (12h40m)

数据来源: Google Flights | 查询时间: 2026-02-25 18:30 UTC
汇率: 1 USD = ¥7.26 CNY = C$1.40 CAD (实时)
⚠️ 价格为实时查询结果，可能随时变动，以实际预订为准。
```

**Format rules:**
- Each flight is numbered and shows airline (中英文) and price in three currencies on the header line
- Price format: `$USD / ¥RMB / C$CAD` (e.g., `$2,164 / ¥15,711 / C$3,030`)
- For connecting flights, total duration goes on a separate line below the header
- Each leg on its own line: `{AIRPORT} {TIME} ✈ {AIRPORT} {TIME} ({LEG_DURATION})`
- Layover line between legs: `⏳ {CITY}{AIRPORT}转机 等待{WAIT_TIME}`
- Flag overnight layovers with `⚠️过夜`
- `+1` suffix on arrival time if it lands the next day

### Currency Conversion

Google Flights returns prices in USD by default. Convert to RMB and CAD:

1. **Get live exchange rates** before formatting output — use web search:
   ```
   web_search: "USD to CNY exchange rate today"
   web_search: "USD to CAD exchange rate today"
   ```
2. **Apply rates** to all USD prices and round to nearest integer
3. **Show all three currencies** on every price line: `$USD / ¥RMB / C$CAD`
4. **Add rate footnote** at the bottom:
   ```
   汇率: 1 USD = ¥X.XX CNY = C$X.XX CAD (实时)
   ```

### Getting Leg Details for Connecting Flights

The initial Google Flights snapshot (efficient mode) only shows summary info (total duration, layover city). To get individual leg details:

1. **Click the flight details button** — each flight in the snapshot has a "Flight details" button (e.g., ref `e33`, `e35`). Click it to expand leg-by-leg breakdown.
2. **Take a new snapshot** after expanding — the expanded view shows each leg's departure/arrival time, duration, and layover wait time.
3. **Repeat for each connecting flight** the user is interested in.

On Trip.com, leg details are available by clicking "Select" on a flight card, which shows the full itinerary breakdown.

**IMPORTANT:** For 转机航班, always expand and list every leg. Users need to know:
- Each leg's departure/arrival airport and time
- Each leg's flight duration
- Layover city, airport, and waiting time
- Whether layover is overnight (过夜转机)

If the user requested specific airlines but they don't appear in results, explicitly state "未找到 {airline} 的航班" and explain why (e.g., no direct/connecting flights on that date).

---

## Common City → IATA Code Mapping

| City | IATA | Airport |
|------|------|---------|
| 上海浦东 | PVG | Shanghai Pudong |
| 上海虹桥 | SHA | Shanghai Hongqiao |
| 北京大兴 | PKX | Beijing Daxing |
| 北京首都 | PEK | Beijing Capital |
| 广州 | CAN | Guangzhou Baiyun |
| 深圳 | SZX | Shenzhen Bao'an |
| 香港 | HKG | Hong Kong |
| 温哥华 | YVR | Vancouver |
| 多伦多 | YYZ | Toronto Pearson |
| 纽约 | JFK | New York JFK |
| 洛杉矶 | LAX | Los Angeles |
| 旧金山 | SFO | San Francisco |
| 西雅图 | SEA | Seattle-Tacoma |
| 伦敦 | LHR | London Heathrow |
| 巴黎 | CDG | Paris CDG |
| 东京成田 | NRT | Tokyo Narita |
| 东京羽田 | HND | Tokyo Haneda |
| 首尔 | ICN | Incheon |
| 新加坡 | SIN | Singapore Changi |
| 台北 | TPE | Taipei Taoyuan |
| 悉尼 | SYD | Sydney |
| 墨尔本 | MEL | Melbourne |

---

## Security & Privacy

**Data that leaves your machine:**
- Flight search queries sent to Google, airline websites via browser

**This skill does NOT:**
- Store search results or personal travel data
- Access payment or booking information
- Auto-book flights
