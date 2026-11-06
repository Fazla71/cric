# Proximity Stream TV – Live Events Web App

Static web app that lists live sports events from API providers and plays their streams (HLS `.m3u8` and DASH `.mpd`, including ClearKey DRM) in the browser. The watch page is laid out like durbinlive.live: a full-width black 16:9 player, a title and info block, server buttons, share, a description tab and "You May Also Like".

## Pages / entry paths
| Path | Purpose |
|---|---|
| `index.html` | Home: hero stats (totals across providers), then a **Live Events** section that lists **only the provider cards** (Provider 1, Provider 2) with live/upcoming/total counts. No events are shown here |
| `provider.html?p=<providerId>` | **The only place events are listed.** One provider's events with search, status and sport filters, countdowns, and chips to switch provider |
| `watch.html?p=<providerId>&id=<eventId>` | Player page (`&slug=` fallback). With no params it opens the first live event |

## How the API is used (Provider 1)
1. `GET https://restless-smoke-4e11.mominulislamm3u8.workers.dev/categories/live-events.txt` returns a JSON array of events: `id, title, slug, eventInfo{teamA, teamB, teamAFlag, teamBFlag, eventCat, eventType, eventLogo, startTime, endTime}, formats[], channelUrl, status`.
2. On the watch page the event's **`channelUrl`** is fetched. It returns `{ streamUrls: [...], related: [...] }`.
3. Each `streamUrls[]` item has these fields:
   - `title`: the server name
   - `link`: the stream URL, optionally followed by `|user-agent=...&Referer=...&Origin=...`
   - `api`: a ClearKey pair `kid:key` (hex) when the stream is DRM protected
   - `type`: `"7"` means DASH + ClearKey, `"0"` means plain HLS
4. The player (Shaka Player 4.12 with mux.js) loads the URL with `drm.clearKeys = {kid: key}`. The live/upcoming/ended status is worked out from start and end times.

## How the API is used (Provider 2)
1. `GET https://falconde-uqdj3fde.manus.space/api/geesports/events` returns a JSON array of `{ event: {...} }` items. Fields: `category, eventName, eventLogo, teamAName, teamBName, teamAFlag, teamBFlag, date` (DD/MM/YYYY), `time` (HH:MM:SS), optional `end_date`/`end_time`, `link_names[]`, `links`/`linkPath` (servers URL), `sourcePath, priority, visible`.
2. The `geesports` adapter in `js/common.js` converts each item to the Provider 1 event shape. The ID is the file name from `sourcePath`. Times are read as UTC; when there is no end time the event is given a 3-hour window.
3. The `links` URL returns an **array** of servers: `{ name, link, scheme, api, tokenApi }`. Servers with an empty `link` that need a server-side `tokenApi` request are skipped, because a browser can't resolve them.

## Adding providers
Edit `js/config.js` and add `{id, name, type, description, api}` to `providers`. `type` chooses the adapter (`cricfy` = Provider 1 format, `geesports` = Provider 2 format). For a new API format, add an adapter in `js/common.js`. `normalizeStreams()` accepts both `{streamUrls:[...]}` and plain server arrays.

## Player features
- HLS and DASH, ClearKey DRM, quality, audio and subtitle menus, PiP, Chromecast, fullscreen
- Switch between servers; the next server is suggested on error; error messages explain DRM, CORS and manifest failures
- Hotkeys: Space/K play or pause, F fullscreen, M mute, arrow keys for volume and seek
- Countdown overlay for upcoming events, with a "Try anyway" button

## Known limitations
- Browsers cannot send custom `User-Agent`/`Referer`/`Origin` headers. Servers that require them, or that block CORS, can fail. A CORS or header proxy would be needed for those.
- `http://` streams are blocked on https pages (mixed content).
- Only ClearKey DRM works. Widevine or PlayReady would need a licence server URL.

## Storage
No database. Everything is fetched live from the provider APIs.

## Files
`index.html`, `provider.html`, `watch.html`, `css/style.css`, `js/config.js`, `js/common.js`, `js/home.js` (provider cards), `js/provider.js` (per-provider event list), `js/player.js`

## Next steps
- Optional stream proxy (for example a Cloudflare Worker) for header-locked or CORS-blocked servers
- More providers and categories (TV channels)
