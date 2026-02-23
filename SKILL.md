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
✈️ {ORIGIN} → {DEST} | {DATE} | {CABIN}舱

| 航空公司 | 出发 | 到达 | 时长 | 经停 | 价格(USD) |
|---------|------|------|------|------|-----------|
| 国泰航空 | 9:30 AM PVG | 11:00 AM YVR | 17h30m | 1 (HKG) | $2,164 |
| ... | ... | ... | ... | ... | ... |

数据来源: Google Flights | 查询时间: {timestamp}
⚠️ 价格为实时查询结果，可能随时变动，以实际预订为准。
```

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
