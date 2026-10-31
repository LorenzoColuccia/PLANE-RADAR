# Plane Radar

A live **ADS-B radar** on a 1.28″ round display: the aircraft flying around you, drawn on a sonar-style screen with their callsign, type and altitude, updated about once a second. Firmware for an **ESP32-C3 Super Mini** with a **GC9A01** round display (240×240).

**Firmware:** [Releases](https://github.com/LorenzoColuccia/PLANE-RADAR/releases) — the full-flash image for a first install by USB, and the signed update files that radars install by themselves over Wi‑Fi when their owner asks (see **Firmware updates**).

Based on [ESP32-Plane-Radar](https://github.com/MatixYo/ESP32-Plane-Radar) by MatixYo (MIT license, see [LICENSE](LICENSE)). Aircraft data from [adsb.fi](https://adsb.fi/).

## What it does

1. **Wi‑Fi setup** (if needed) — captive portal on AP **`PlaneRadar-Setup`**
2. **Radar** — live aircraft from [adsb.fi](https://opendata.adsb.fi/) on a sonar-style grid
3. **Firmware updates** — from releases you publish on GitHub, signed by you, installed only when the radar's owner presses *Install update* on its page (see **Firmware updates**)

After Wi‑Fi is saved, the device reconnects automatically. The radar display and the BOOT button run in the main loop; ADS-B updates (about once a second) are fetched by a separate background task, so a slow network never delays the button. A radar on a shelf is rarely watched, so if something wedges the device notices and restarts itself, and logs why (see **Reliability**).

## Controls (BOOT, GPIO 9, active LOW)

| Action | Effect |
|--------|--------|
| **Short tap** | Cycle range preset (5 → 10 → 15 → 25 km, then back to 5); saved to flash |
| **Hold 3 s** | Clear Wi‑Fi, location, and units; reboot into setup portal |

The hold works whenever the firmware is running: on the radar, on the "Connecting" screen and at the setup portal. Don't hold BOOT while powering on or pressing RESET: on the ESP32‑C3 a low GPIO 9 at reset starts the ROM download mode instead of the firmware (which is what you want for flashing, see **Build**).

**How a press is handled.** It is built so that a tap is not lost whatever the device happens to be doing:

- A GPIO interrupt time-stamps every edge of the button into a small queue and takes no decisions. The main loop feeds the queued edges to `util::ButtonTracker` (`include/util/button_tracker.h`, covered by host tests) and also samples the pin on every pass, as a safety net for an edge the interrupt delivered late (the CPU can't run an interrupt while flash is being written, for example).
- Contact bounce is merged, not counted: a release followed by a new press within 12 ms (`kBootBounceMs`) is the same press, so chatter in the middle of a press can neither split it into two taps nor make a real press look too short. Presses under 20 ms (`kBootTapMinMs`) are noise.
- Every tap counts. A burst of taps steps the range once per tap and costs a single redraw and a single fetch request.
- A tap is applied on the next pass of the main loop, at most one screen redraw later: the loop never waits for the network (the fetch runs on its own task), and the slow parts that remain, the Wi‑Fi connect and the setup portal's wait, apply taps from inside their own wait loops.
- The new range is drawn at once from the data already on screen, and the fetch task is asked to fetch for it immediately (never closer than 1.1 s after the previous request started), so aircraft that were beyond the old fetch radius show up as soon as the API's rate limit allows.
- `short taps` in the health log counts the presses rejected as too short. If it keeps growing while nobody touches the button, suspect a noisy contact or a loose wire.

## Wi‑Fi setup portal

**First-time setup** (no saved Wi‑Fi):

1. Connect to **`PlaneRadar-Setup`**
2. Open **`http://plane-radar.local`** (preferred) or **`http://192.168.4.1`** — both are shown on the yellow setup screen; captive portal may open automatically
3. Set home Wi‑Fi, then save

**Reconfigure anytime** (after the device is on your network):

1. Open **`http://<device-ip>`** or **`http://plane-radar.local`**. The radar shows both for 8 seconds after it connects at power-on (`kAddressScreenMs`); the router's device list and the serial log have the IP too. On many routers **`http://plane-radar`** works as well: the radar gives the router that name when it asks for an address
2. Change Wi‑Fi, location, units, or runway overlay; save

The same portal runs on the setup AP and on the device’s LAN IP while connected to Wi‑Fi. mDNS hostname is `plane-radar` → **plane-radar.local** (`kPortalHostname` in `config.h`). Some clients resolve `.local` slowly; use the IP if needed. Open the page by that name or by the IP: saving from any other host name is refused (see **Security notes**).

**Custom fields** (stored in NVS):

| Field | Purpose |
|-------|---------|
| **Latitude / Longitude** | Radar center and ADS-B query position (defaults in `config.h` until set) |
| **Display distances in miles** | Ring scale label in **mi** instead of **km** (e.g. `6mi` vs `10km`) |
| **Show airport runways** | Major-airport runway overlay on the radar (off to hide) |

After a reset, the device reboots and shows the setup screen immediately (no “Connecting” loop on stale credentials).

**Router still down after a power cut?** If a network is saved but can't be reached at boot, the setup portal opens but closes by itself after 2 minutes without use (`kWifiPortalTimeoutSavedSec`), and the device goes back to reconnecting in the background, so a radar that boots before the router comes back recovers on its own. A phone connected to the setup AP keeps the portal open. With no network saved, the portal waits for you indefinitely (`kWifiPortalTimeoutSec = 0`).

## Radar display

### Grid

- Dark blue background, subdued green rings and crosshairs
- White **N / S / E / W** at the bezel; range label on the **east** spoke (ring 3 = ¾ of outer radius)
- White center dot

Layout and colors: `include/ui/radar_theme.h`.

### Range presets

| Ring 3 label | Outer radius (aircraft scale) |
|------------|-------------------------------|
| 5 km / 3 mi | ~6.7 km |
| 10 km / 6 mi | ~13.3 km (default) |
| 15 km / 9 mi | ~20 km |
| 25 km / 16 mi | ~33.3 km |

Preset and miles/km choice persist across reboot (`planeradar` NVS namespace).

### Runways

- Major airports from OurAirports (`large_airport`): 1112 airports and 1705 runways, every open runway strip (helipads excluded)
- Teal runway lines with one ICAO label per airport (e.g. `KJFK`); toggle in the Wi‑Fi setup portal
- Lines are cut exactly at the edge of the radar disc, including a runway that starts outside it, crosses it and ends outside again (`util::clip::segmentToDisc`)
- Cheap to redraw: the airport list is first narrowed by two integer comparisons per airport (`util::geo::BoxE7`) and only the survivors get a distance check, so a redraw does no trigonometry per airport
- Update the embedded list: `python3 scripts/build_large_airports.py` (downloads OurAirports' `airports.csv` and `runways.csv`; `--csv-dir DIR` reads local copies, `--out-dir DIR` writes somewhere else for a dry run). The script checks the CSV columns and that the result is about the size of the real dataset before it replaces anything, drops airports with no drawable runway, and writes both files only once both are complete. `scripts/test_build_large_airports.py` tests it.

### Aircraft

- **Inside the outer ring** — red heading triangle, magenta speed vector (cut exactly at the ring), callsign / type / altitude tags; altitude is shown in **km** (e.g. `10.7 km`, see `kAltitude*` in `config.h` for meters or feet)
- **Outside the ring** (still within ADS-B fetch) — small **red dot on the screen rim** at the correct bearing (direction cue; not distance-accurate past the ring)
- **Tags** — placed toward the **center**: west (left) → tag on the **right** of the symbol; east (right) → tag on the **left**

As range decreases (or aircraft approach), targets move inward; beyond-ring dots become full symbols when they cross the outer ring. Positions use one shared flat-earth projection (`util::geo`, `ui::radar::Projection`) that also works across the ±180° date line.

### How a frame reaches the screen (and why it doesn't flicker)

A new picture is drawn after every ADS-B answer (about once a second). It is composed off-screen and only then sent to the panel, so the panel never shows a half-drawn or erased screen.

- **The old way, and its flaw.** Upstream composes the frame in one 240×240 sprite: 115 KB, of the ESP32-C3's ~320 KB of RAM. With Wi‑Fi up and a TLS session open there is not always a block that large left, and fragmentation can take it away after hours of running. When the sprite could not be allocated the code fell back to drawing straight on the panel, starting with a full black clear — which would show as a black flash on every refresh. This is what the code analysis points to as the cause of a reported flicker every few seconds; it has not been confirmed on a board, so the log lines under **Reading the serial log** are there to tell.
- **Now: strips.** `radarDisplayInit()` (called at boot, before Wi‑Fi starts, while the heap is still in one piece) allocates one 240×40 strip, 19 KB, and keeps it for good, so nothing can fragment it away later. A frame is composed one strip at a time — clear the strip, draw everything that reaches it, push it to the panel — six pushes for a frame. Each strip is complete when it is pushed, and the panel is never cleared, so there is nothing black to see. The price is that the drawing code runs once per strip (everything outside the strip is skipped quickly) and the screen refreshes as a fast top-to-bottom sweep instead of in a single instant; the serial health line reports how long a frame takes (`screen ... ms`).
- **If even 19 KB is not there at boot** the renderer tries 30, 24, 16 and 8 rows (all divide 240), and only then draws on the panel as before. That path leaves one line in the log (`radar: no memory for a frame strip`) and shows up as `on-panel N` in the health line.
- **Same picture, whatever the strip height.** The drawing code works in whole-screen coordinates; `ui::StripCanvas` (`include/ui/strip_canvas.h`) shifts them for the strip and the sprite clips the rest. One detail: LovyanGFX's `drawWideLine` misclips lines at the edge of the surface it draws on (it passes a corner where a width and height are expected), which a strip has everywhere, so `StripCanvas::drawWideLine` is the library's algorithm with the clipping fixed. `test/render/` draws 240 scenes through the real LovyanGFX on a simulated panel and checks that every strip height produces exactly the same pixels as the whole-frame case; when the strip renderer was written, its output was also compared pixel for pixel with the old full-sprite renderer's (see **Testing**).

### ADS-B

- Source: `https://opendata.adsb.fi/api/v3/lat/<lat>/lon/<lon>/dist/<nm>`
- Fetch radius: `ui::radar::fetchRadiusKm()` — scales with the active preset to roughly the screen edge (so rim dots have data)
- **Privacy:** the position sent is rounded to 0.01° (about 1.1 km, `kAdsbQueryStepDeg`) and the radius is padded by the most the rounding can move the center, so every aircraft on screen is still inside the queried circle. The exact coordinates of the radar, very likely your home, stay on the device. The API still sees your IP address, and so learns your town. Set `kAdsbQueryStepDeg` to `0` to send exact coordinates.
- **Cadence:** a request starts every `kAdsbFetchIntervalMs` (1.1 s) after the previous one *started*, from a dedicated background task (`services/adsb_task.cpp`), see **Reliability**. Counting from the start means the screen updates every 1.1 s whatever the round trip took, as long as it took less than that; a slower round trip (a TLS handshake on a weak signal, say) is followed by the next request at once. 1.1 s is the fastest the API's 1 request/second limit allows with a 10% margin; the log's `request N ms (max M)` field shows how long the round trips really take on your network (see **Reading the serial log**). A range change (BOOT tap) wakes the task immediately instead of waiting for the next tick; after a Wi‑Fi outage the first fetch follows as soon as the link is back. Two requests never start closer than `kAdsbMinRequestSpacingMs` (1.1 s) apart, whatever the timing of taps.
- **When it fails:** after a failed request the interval (counted from the end of the failed request) doubles (1.1 s, 2.2 s, 4.4 s, ...) up to `kAdsbMaxBackoffMs` (60 s) and snaps back on the first success. On HTTP 429, or a 503 that says for how long, the `Retry-After` header is honored (seconds form; capped at `kAdsbRetryAfterMaxMs`, 5 min, so a bogus value can't park the radar for hours); with no usable header a 429 waits `kAdsbRateLimitedDefaultMs` (5 s: at this cadence a stray 429 is two requests that landed slightly under a second apart, and repeated ones stretch the interval by the doubling above). If `rate-limited` keeps showing up in the health line, raise `kAdsbFetchIntervalMs` (3000 was the original value). Before every request the largest free heap block is checked against `kAdsbMinHeapBlockBytes` (20 KB, enough for the TLS buffers); if it is smaller the request is skipped instead of failing half-way inside the TLS stack.
- **Parsing is streamed, one aircraft at a time** (`util::BufferedStreamReader` → `util/adsb_stream.h`): the body is never held whole, and neither are all the aircraft. Each entry is read into one small ArduinoJson document (about 1.2 KB) through a [filter](https://arduinojson.org/v7/how-to/reduce-memory-usage/) that keeps only the dozen fields the radar draws, converted, and then discarded; the 64 aircraft nearest to the radar's center (`kMaxAircraft`) are kept. Memory use is therefore the same over a quiet field and over a busy airport (a busy area's response is tens of KB, ~28 KB for 31 aircraft around Amsterdam at the 25 km preset), which is where it matters most.
- **Bad data is not trusted:** an `ac` (or `aircraft`) array is expected; `"ac": null` is an empty sky, and a reply with no array at all is a failed fetch. Every number is checked before it is used: latitude and longitude must be real coordinates, speed (0–5000 kt), heading and altitude must be plausible, and NaN, infinity and absurd values count as missing (the next field that can say the same thing is used). An entry without a usable position is skipped, and more than eight unreadable entries make the whole reply a failed fetch. The nearest-64 ranking measures longitude across the date line correctly.
- Ground aircraft hidden by default (`kAdsbShowGroundAircraft`)
- The range-scale label (e.g. `10km`) turns **amber** if there's been no successful fetch for `kAdsbStaleMs` (30 s: many missed cycles, so one slow request doesn't trigger it; counted from the first attempt if none has succeeded since boot) — a signal that what's on screen may be out of date (Wi‑Fi trouble, or an adsb.fi outage) rather than an empty, quiet sky. It turns back to normal the moment a fetch succeeds again. A failed fetch cycle still redraws, so the indicator gets its chance to show.
- Requests carry `User-Agent: ESP32-Plane-Radar/<version>` and `Accept: application/json`, so the API's operator can tell a well-behaved client from a stuck one.
- The device never validates adsb.fi's TLS certificate (`WiFiClientSecure::setInsecure()`). See **Security notes** below for why, and the trade-off that implies.
- adsb.fi's public endpoint is rate limited to **1 request/second** and is for personal, non-commercial use (see [adsbfi/opendata](https://github.com/adsbfi/opendata)); the default 1.1 s interval (start to start, never shorter) stays just under that. Their terms also ask to cite adsb.fi with a link to their site. **Selling radars that use it is commercial use**: their terms say to contact them for that, so do before you sell (or switch provider, see below). If adsb.fi ever changes terms or goes down, `services/adsb_client.cpp` is the one place a different provider (adsb.lol, airplanes.live — both are drop-in-compatible ADSBExchange-style APIs) would be swapped in.

## Reliability

- **The ADS-B fetch runs on its own task, off the main loop** (`services/adsb_task.cpp`). `services::adsb::fetchUpdate()` blocks for as long as the HTTPS round trip takes (TCP connect + TLS handshake + request + response, anywhere from ~100 ms to a couple of seconds, and the underlying HTTPClient/WiFiClientSecure calls can't be interrupted from the outside). The loop that handles the button, the display, WiFiManager and OTA never calls anything that touches the network, so a slow or stuck request can't delay it.
  - The two tasks hand aircraft data over through a small mutex-protected snapshot (`services::adsb::snapshotAircraft()`), so the display never reads a batch mid-write.
  - `services::adsb_task::requestImmediateFetch()` (a FreeRTOS task notification) wakes the fetch task for a range change.
  - The task has its own 12 KB stack and subscribes to the same task watchdog as the main loop.
  - The only remaining blocking loops are the Wi‑Fi connect/reconnect and the setup portal's wait (`services/wifi_setup.cpp`); they drain queued BOOT taps and the OTA handler from inside their wait loops (`wifiSetPollFn()`).
- **Every network wait is bounded under the watchdog** (`services/adsb_client.cpp`): DNS is its own step (watchdog fed before and after), TCP connect 2 s, TLS handshake 5 s (the library default is 120 s), response headers 5 s, and the body read feeds the watchdog while it waits and gives up after 5 s without a byte or 15 s in total.
- **Timers are safe across the `millis()` wrap** (every ~49.7 days of uptime): everything compares elapsed time (`now - earlier >= span`) instead of storing an absolute deadline, which at the wrap would fire back-to-back requests or stop fetching for another 49.7 days. The tests cover it, including a 2-minute simulation straddling the wrap.
- **Watchdog** — a 20 s task watchdog (`hardware::watchdog`, `kWatchdogTimeoutSec`) reboots the device if the main loop or the fetch task ever truly locks up. Every normal long wait (Wi‑Fi connect, the setup portal, an in-progress OTA transfer, the ADS-B fetch) feeds it from inside its own loop.
- **Self-healing** (`services/health.cpp`) — a watchdog can't see a device that runs fine and does nothing useful, so the health module restarts it when
  - Wi‑Fi has been up for an hour (`kAdsbStallRestartMs`) and not one ADS-B fetch has succeeded in that time (a wedged network stack; a genuine adsb.fi outage just costs one reboot per hour), or
  - ten fetches in a row had to be skipped for lack of contiguous heap (`kLowHeapRestartSkips`), meaning the heap is fragmented beyond use.

  Both leave a `health: restarting — <reason>` line in the serial log. The Wi‑Fi side heals itself too: a dropped connection is retried every 15 s, and a saved network that is unreachable at boot (router still starting) only holds the setup portal open for 2 minutes, see above.
- **Logging can't stall the button.** The USB serial port blocks each write for up to 100 ms once a PC has enumerated it but nothing is reading (cable plugged into a computer, no serial monitor open). The TX timeout is set to 0 (unread log lines are dropped) with a 1 KB buffer, so a `Serial.printf` on a BOOT tap returns immediately.
- **OTA pauses the ADS-B fetch** for the duration of a transfer, so the update isn't competing with a TLS session (~40 KB of heap) and its traffic; it resumes automatically if the update fails.
- **Debounced settings writes** — cycling the range preset doesn't write to flash on every tap. A change is committed ~1 s after taps stop (`kRangeSaveDebounceMs` in `radar_range.cpp`), so a burst of presses becomes one flash write. The trade-off: a range change made in the last second before a power loss won't have been saved yet.

### Reading the serial log

`pio device monitor` (115200 baud). At boot, one line says what the firmware is and why the chip last reset, which tells a power cut from a crash or a hang:

```
Plane Radar 1.3.0, last reset: power-on
```

(other values: `software restart`, `CRASH (panic)`, `HANG (task watchdog)`, `BROWN-OUT (weak power supply)`, ...). Next come the screen buffer line and two memory snapshots, taken while booting: right after the display is set up and once Wi‑Fi has connected. They show how much room the network took and how big the biggest free block is (illustrations, not measurements):

```
radar: frame strip 40 rows, 19200 bytes
health: heap after display init: free 262000, largest block 110000
health: heap after WiFi setup: free 118000, largest block 69000
```

If the first line is `radar: no memory for a frame strip - drawing straight on the panel` instead, the screen will flicker on every refresh; please report it together with the two memory lines. Then, once a minute (`kHealthLogIntervalMs`), a status line:

```
health: up 0d03h12m | heap free 118000 min 101000 largest 69000 | wifi -61 dBm | fetch 3812/3830 ok http 16 bad-body 2 | request 640 ms (max 1900) | screen 40-row strips 62 ms (max 71) on-panel 0 | last http 200 | stack free loop 3180 adsb 2480 | short taps 0
```

| Field | Meaning |
|-------|---------|
| `up` | Uptime (days, hours, minutes) |
| `heap free / min / largest` | Free heap now, the lowest it has been since boot, and the largest single free block (a TLS handshake needs `kAdsbMinHeapBlockBytes`). A `min` that keeps shrinking over days is a leak; a small `largest` next to a big `free` is fragmentation |
| `wifi` | Signal strength in dBm (−70 or stronger is comfortable; below about −80 expect trouble), or `wifi down` |
| `fetch ok/attempts` + counts | Successful requests out of all attempts since boot, then how the others ended: `dns`, `connect`, `http`, `rate-limited`, `bad-body`, `low-heap` |
| `request` | How long the last successful ADS-B request took (DNS, TLS handshake, response) and the longest since boot. If it stays under `kAdsbFetchIntervalMs` (1100) the screen updates every 1.1 s; if it is above, the radar updates as often as the server answers |
| `screen` | Rows per frame strip in use (`0-row` = no strip, drawing straight on the panel), how long the last whole-screen redraw took and the longest one since boot, and how many frames had to be drawn straight on the panel for lack of a strip (`on-panel`; should stay `0`) |
| `last http` | HTTP status of the latest request (`200` is healthy; negative values are the HTTP client's own errors, such as a timeout) |
| `stack free` | Smallest amount of stack (bytes) ever left over in the main loop and in the fetch task; close to 0 means a stack overflow is near |
| `short taps` | Button presses rejected as too short (see **Controls**) |

## Firmware updates (for radars you sell)

A radar updates itself over its WiFi, from releases you publish on GitHub, **only when its owner asks**: nothing changes on a customer's radar until they press a button. There is no server to run: GitHub stores the files for free, and the radar itself serves the page that installs them.

### What your customer does

1. Open **`http://plane-radar.local`** on a phone or computer on the same WiFi, and press **Firmware update** (or go straight to `http://plane-radar.local/fw`). If that address does not open (`.local` names depend on the phone or computer and on the router: some Android phones, phones with a VPN on, some Windows set-ups and routers that filter multicast cannot use them), unplug the radar and plug it back in: for 8 seconds after it connects, its screen shows its address, *"http://192.168.1.23"*; type exactly that into the browser's address bar. On many routers `http://plane-radar` works too.
2. The page checks GitHub and says either *"This radar is up to date"* or *"Version 1.3.1 is available"*, with your one-line note on what changed.
3. **Install update** → the radar's screen shows the progress and it restarts after about a minute. The page then waits for the new version to prove itself (about two minutes, see below; *"checking that it works properly: keep it plugged in"*) and only then says *"Done! The radar now runs version 1.3.1"*, or that the radar went back to the previous version.

So when a customer reports a problem: fix it, publish the release, and tell them to open the page and press the button.

### What the radar checks (`services/fw_update.cpp`, `util/fw_release.h`)

- **Only releases you signed.** A release is two files on a GitHub release: `manifest.json` and `plane-radar-update.bin`. The manifest is a short statement (board, version, size, SHA-256 of the image, notes) signed with your private key (ECDSA P-256). Every radar has your public key built in and refuses anything it does not verify — whoever made it, wherever it came from (a hijacked network, a fake file, a forged release). The signature, not the TLS connection, is what makes a release trustworthy, so the radar does not need GitHub's certificates.
- **Only newer versions** (no downgrades, not even to your own older releases), only for its board (`esp32c3-supermini`), and only if it fits the spare app slot.
- **The image is checked while it downloads**: a file that is not an ESP32-C3 application (the full-flash image, say) is refused before a byte reaches the flash; the size and SHA-256 must be exactly the signed ones before the new firmware is activated. A failed or interrupted download changes nothing: the radar keeps running its current firmware and shows *"Update failed"* for a few seconds.
- **The new firmware must be able to update itself in turn.** Every firmware carries one line of text about itself, `PLANE-RADAR-FW|<version>|<board>|<update repository>|<public key>|`. The radar reads it from the image as it downloads and installs the image only if it is the version the signed release names, for its board, and able to find and check its own updates: built with a valid public key (a real P-256 point, not just 130 hex digits: one mistyped digit makes a key that can never verify anything) and with the **same update repository** as the running firmware (compared ignoring case). A release built by mistake without them, or with a typo in them, would leave radars that could never update again. There is never a need to change the repository: one renamed or transferred on GitHub keeps answering at its old name. The signing tools read the same line and refuse such a firmware before it is signed (they cannot know the radars' repository, so they show it: check it). (A firmware also contains the marker `PLANE-RADAR-FW|` a second time, on its own, as the constant its scanner looks for; every reader skips occurrences that are not a complete line.)
- **A bad release cannot strand a radar.** The update goes into the second app slot (two-slot table, `partitions/plane_radar.csv`). After the restart the new firmware is *on trial*: it confirms itself once it has been on WiFi for 2 minutes in all (`kUpdateConfirmAfterMs`), long enough to have fetched, drawn and served its page many times over. If it crashes or hangs before that, or is still off WiFi 10 minutes after the restart (`kUpdateTrialMs`), the bootloader goes back to the previous firmware, and the page says so. Any restart during the trial counts, a power cut included: the radar then simply runs its previous version, and the update can be installed again. While a firmware is on trial it does not install another one.
- **The page itself** only accepts its requests from the radar's own page (`util/request_guard.h`: no other web site can make a browser press the button), can't be framed by another site, loads nothing from the internet, and works on a phone. Anyone on the customer's network can press *Install update*, but all it can ever install is your newest signed release.
- The check and the download run on the network task that fetches aircraft (one TLS session at a time), so the button and the screen stay responsive.

### One-time setup

1. **A public GitHub repository for releases**, e.g. `your-name/plane-radar-firmware`. It can be empty apart from a README; it only holds the release files (it can also be the repository with this source code).
2. **A signing key.** Open `tools/release-signer.html` (double-click it: it runs offline in the browser, nothing is uploaded) and press **Create a new key**. It downloads `plane-radar-signing-key.json` and shows the **public key** (130 hex digits). Keep the key file private and backed up: it is the only thing that can sign updates for your radars. If it is lost, radars can only be updated by USB; if it leaks, someone could sign updates (they would still need to publish them in your repository). `scripts/sign_release.py keygen` does the same from the command line.
3. **Build the firmware with both**, in `include/config.h` (`UPDATE_GITHUB_REPO`, `UPDATE_PUBLIC_KEY`) or at build time:
   ```bash
   UPDATE_GITHUB_REPO=your-name/plane-radar-firmware UPDATE_PUBLIC_KEY=04… pio run -e supermini
   ```
   Then flash it by USB (see **Build**) on every radar. A build without them says *"Updates are not set up"* on the page; the boot log says which one is missing. The repository is final: every later release must look for updates in the same one (radars refuse one that does not).

   *This source already has them in `include/config.h`: releases from `github.com/LorenzoColuccia/PLANE-RADAR`, signed with the key whose fingerprint is `2570 CEBB 15C4 FB5A`.*

### Publishing an update

1. Build the new version: bump `FIRMWARE_VERSION` in `config.h` (or `FIRMWARE_VERSION=1.3.1 pio run -e supermini`). Try it on your own radar first (USB).
2. Sign it: in `tools/release-signer.html` open your key file, choose **`firmware.bin`** (`.pio/build/supermini/firmware.bin`, *not* the full-flash image), add a one-line note if you like, press **Sign release**. You get `manifest.json` and `plane-radar-update.bin`. The tool reads the version and the update repository from the firmware (check that the repository is yours), and refuses: the wrong file (the full-flash image), a file that is cut short or damaged (it checks the image's own checksum and SHA-256), a development build or a version with leading zeros, a firmware built without update settings or with a key that is not a valid key, and — unless you tick that moving radars to another key is the point — a firmware that trusts a different key than yours (both keys are shown by fingerprint). Same on the command line:
   ```bash
   python3 scripts/sign_release.py sign --key plane-radar-signing-key.json \
     --firmware .pio/build/supermini/firmware.bin --notes "Fixes the screen flicker"
   ```
3. On GitHub: **Releases → Draft a new release**, tag **`v1.3.1`** (exactly `v` + the version: that is where radars look for the image), attach the two files, leave *pre-release* unticked, **Publish**. (The signing tool links straight to that page.) Radars see the newest non-pre-release release; a pre-release is invisible to them.
   - GitHub cannot make a release in an **empty repository**: add a file first (*Add a README*).
   - The two files must be **attached to the release**, under *Attach binaries*, with exactly these names (a browser that already had a `manifest.json` in its downloads may have saved `manifest (1).json`: rename it). Wait until both have finished uploading before pressing *Publish*. A release that only shows GitHub's own *Source code* archives has no files attached: radars then say *"the newest release on GitHub (v1.3.1) has no manifest.json attached"*. Edit the release (pencil icon), attach the two files, *Update release*.

`scripts/sign_release.py verify --manifest manifest.json --firmware plane-radar-update.bin --public-key 04…` checks a release the way a radar would. With the source on GitHub, the release workflow can do steps 1–3 by itself on every `v*` tag (see **CI and releases**).

### Reading the serial log

```
firmware: PLANE-RADAR-FW|1.3.1|esp32c3-supermini|your-name/plane-radar-firmware|04…| from app1 - new, on trial until it has run on WiFi for a while
firmware: updates from github.com/your-name/plane-radar-firmware, signed releases only
firmware: new firmware confirmed
update: checking https://github.com/your-name/plane-radar-firmware/releases/latest/download/manifest.json
update: newest release 1.3.2 (running 1.3.1): can be installed
update: downloading https://github.com/your-name/plane-radar-firmware/releases/download/v1.3.2/plane-radar-update.bin (1357856 bytes)
update: 1.3.2 installed, restarting
update: network task stack min free 5120 bytes
```

(An illustration, not a recording.)

### Developer uploads (ArduinoOTA)

For your own bench radar there is also [ArduinoOTA](https://docs.espressif.com/projects/arduino-esp32/en/latest/ota_updates/ota_web_updater.html) (`espota`, port 3232): it accepts **any** firmware from anyone who knows its password, so it is **off** unless a password is set at build time, and must never be in a radar you sell:

```bash
PLATFORMIO_BUILD_FLAGS='-D OTA_PASSWORD=\"your-password-here\"' pio run -e supermini
```

With it on, upload with a second environment in `platformio.ini`:

```ini
[env:supermini_ota]
extends = env:supermini
upload_protocol = espota
upload_port = plane-radar.local
upload_flags = --auth=your-password-here
```

```bash
pio run -e supermini_ota -t upload
```

## Security notes

- **The setup portal only does what the setup page needs.** WiFiManager registers every one of its pages whether or not its menu shows them, and none of them ask who is calling, so `services/wifi_setup.cpp` puts a guard in front of them:
  - `/update` and `/u` (WiFiManager's unauthenticated firmware upload), `/erase`, `/restart`, `/exit`, `/close` and `/info` always answer **403**. Firmware updates go through the radar's own update page, which only installs signed releases (see **Firmware updates**).
  - `/wifisave` and `/paramsave`, the two requests that change the Wi‑Fi login and the radar's position, are accepted as a **POST only**, and only if the `Host` header names this device (an IPv4 address, or `plane-radar` bare or with `.local`, `.lan` or another private-network suffix) and the browser doesn't say the request comes from another site (`Origin`, `Sec-Fetch-Site`). That stops a web page you happen to have open from reconfiguring the radar through your browser (cross-site request forgery), including through a hostile domain name pointed at the device's IP address (DNS rebinding). Tools that send neither header, such as `curl`, still work. The check is `include/util/request_guard.h`, with tests.
  - So open the setup page as `http://plane-radar.local` or `http://<device-ip>`; saving from any other host name answers 403 with a hint.
- **The setup page has no password.** While the radar is on your network its page is reachable by anyone on that network, who can read the saved location (the form is pre-filled with it) and change the settings. That is fine on a home network; don't connect the radar to a network you don't trust. The setup access point is open as well, as captive portals are, but it only exists when there is no usable Wi‑Fi: first setup, after a reset, and for 2 minutes at boot if a saved network can't be reached.
- **The ADS-B query is coarsened** (see **ADS-B**): about 1 km of precision instead of the exact coordinates. The API still sees your IP address.
- **ADS-B fetch uses `setInsecure()`** — no TLS certificate validation against adsb.fi. This is intentional, not an oversight: pinning a specific certificate would break every fetch the day that (short-lived, Let's-Encrypt-style) certificate rotates, with no way for the device to recover short of a firmware update, and validating against a CA bundle needs the correct date, which the device only has after an NTP round trip it doesn't otherwise make. The data being fetched is public and read-only, so the realistic risk is a MITM on the local network path feeding the device forged aircraft positions — annoying, not dangerous; every value is range-checked on the way in, no more than 64 aircraft are kept, and the read ends at the end of the aircraft list or after 15 s. If you want certificate validation, `WiFiClientSecure::setCACert()` with the appropriate root CA is the place to add it; just be prepared to update it if adsb.fi's CA changes.
- **Firmware updates are signed.** A radar installs only releases signed with the seller's private key, newer than the one it runs, for its board; see **Firmware updates** for what is checked and why the TLS certificate of GitHub does not need to be. The private key is the one secret in the whole system: keep it off any machine you don't trust, out of the repository (`.gitignore` covers `*signing-key*.json`), and in CI only as a repository secret.
- **ArduinoOTA is off** unless a password is chosen at build time; with it on, anyone on the network who knows the password can flash anything, with no rate limiting or TLS — fine on a developer's bench, never on a radar that is sold.

## Testing

The pure logic lives in `include/util/` (plus `src/util/` and a few header-only helpers in `include/ui/`) with no Arduino/ESP-IDF dependency and has host-side tests in `test/native/` — no PlatformIO or ESP32 toolchain needed. One script builds and runs all of them:

```bash
scripts/run_host_tests.sh                  # 64-bit, 32-bit and sanitizer builds + the generator's tests
scripts/run_host_tests.sh --quick          # 64-bit only
scripts/run_host_tests.sh --require-32bit  # fail instead of skipping if -m32 is unavailable
```

It needs `g++` (and `g++-multilib` for the 32-bit build) and `python3`. ArduinoJson, header-only, is taken from `$ARDUINOJSON_DIR`, from PlatformIO's `.pio/libdeps`, or cloned at the pinned tag. Every build uses `-Wall -Wextra -Werror`, in three flavours: normal 64-bit; 32-bit (`-m32`, the ESP32's pointer and `int` size, which is where the memory figures come from); and AddressSanitizer + UBSan, which catch memory errors and undefined behaviour the plain build can miss.

| Test | Covers |
|------|--------|
| `test_format.cpp` | Altitude/range label formatting, lat/lon parsing and validation |
| `test_fetch_schedule.cpp` | When the fetch task may start a request: cadence (start to start, with the firmware's 1.1 s values fuzzed against random round trips, failures and taps), rate-limit floor, immediate requests, backoff after failures, behavior across the `millis()` wrap |
| `test_adsb_parse.cpp` | The real response path: synthetic adsb.fi responses streamed through `util::BufferedStreamReader` and `util/adsb_stream.h` into ArduinoJson and `util::adsb_parse` — arbitrary chunking, a connection dropped mid-body, a stalled server, a slow-but-steady link, the clock wrap during a wait, empty/missing/null `ac`, bad values, the 64-aircraft cap, nearest-first ranking across the date line, and memory use |
| `test_http_helpers.cpp` | `Retry-After` parsing and the rounded query position and radius sent to the API |
| `test_request_guard.cpp` | Which setup-portal requests are trusted (`Host`, `Origin`, `Sec-Fetch-Site`) |
| `test_button_tracker.cpp` | Tap/hold/debounce logic: bounce, chatter mid-press, quick taps, long holds, missed edges, the clock wrap |
| `test_geo.cpp` | Flat-earth offsets, date-line wrap, poles, and the integer pre-filter for the airport scan |
| `test_disc_clip.cpp` | Clipping a line to the radar disc, fuzzed against a brute-force walk along the segment, including coordinates far off screen |
| `test_projection.cpp` | Latitude/longitude → pixels: orientation, scale, rounding, date line |
| `test_runway_scan.cpp` | The real embedded airport table: the pre-filter never drops an airport that is in range, for every airport used as the radar position at every range |
| `test_fw_release.cpp` | Firmware updates, the checks: versions, the signed statement (every malformed or ambiguous form refused), `manifest.json`, release URLs and the GitHub hosts a download may be redirected to, the image header (the full-flash file refused), the streaming receiver in chunks of any size (too long, cut short, one bit changed, a failing flash), the firmware's identity line wherever chunk boundaries fall (the marker also present on its own, as in a real build; the most telling reason when no line is valid; the longest identity a build can have), and ECDSA P-256 on mbedTLS 2.28 — the library inside the ESP32 core — against signatures made with an independent implementation (pyca/cryptography), including the high-*s* form browsers may produce, degenerate signatures and bad keys |
| `test/fw_update/` | The update service itself (`services/fw_update.cpp`, the real file) with fakes for the network, GitHub's redirects, the flash writer, the OTA partition state and the web server, one process per scenario: check, install and restart; no downgrades; another seller's signature; an edited manifest; another board; a release too big; no release; a redirect off GitHub or in a loop; a new firmware without update settings, with a key off the curve, looking for updates in another repository (the same one in other letter case is fine), or laid out like a real build (the marker twice); DNS failure; low memory; a corrupt, wrong, truncated, stalled or mis-sized download; flash and activation failures; a retry; the page's request guards; the rollback trial (confirmed on WiFi, rolled back after 10 minutes, nothing to roll back to, nothing to do after a USB flash). Also run by `scripts/run_host_tests.sh` (`scripts/run_fw_update_tests.sh`) |
| `scripts/test_build_large_airports.py` | The runway-data generator: CSV parsing, helipads and unusable rows, column checks, size sanity floors, atomic writes (a bad download never replaces good files), and the committed data files themselves (counts match, sorted, positions in range, no degenerate runways) |

All of these run on every push in CI (`native-tests` job in `.github/workflows/build.yml`) and before every release. The update test needs `gcc` too: it builds mbedTLS 2.28.4 (the version in the ESP32 core) for the host once, into `.pio/host-test/` (`scripts/lib_mbedtls.sh`).

### Web tests (update page and release tools)

```bash
scripts/run_web_tests.sh
```

Runs the parts of the update system that live in a browser or on the seller's computer, in a real browser (headless Chromium through Playwright; `pip install cryptography playwright && python3 -m playwright install chromium`):

- **The radar's update page** (`src/services/fw_update_page.cpp`) against a stand-in for the radar (`test/web/mock_radar.py`): the whole install with the radar disappearing during its restart and coming back with the new version, "Done" only once that version has been confirmed, a rollback reported as such (at once, or during the trial), up to date, a failed check or install with its reason, the confirmation dialog, a radar without updates set up or without WiFi, no loop of refused checks, release notes shown as text (never HTML), a phone-width screen in dark mode, no script errors and nothing loaded from the internet.
- **The signing tool** (`tools/release-signer.html`, opened from disk): creating and opening keys, signing, the release link, refusing the full-flash image, older or development builds, versions with leading zeros, builds without update settings or with a key off the curve, files cut short or damaged, and damaged key files, a deliberate tick (for that one firmware only) to sign a firmware that trusts another key, the same reason as the command-line tool for every refusal, the identity line found past the marker's other occurrence (giving the same reason as the command-line tool when there is no valid line), and the real `firmware.bin` from the last PlatformIO build. Every release it signs (with WebCrypto) is then checked by the radar's own code compiled for the host (`test/web/verify_release.cpp`) — and tampered ones must be refused; keys move between the browser tool and `scripts/sign_release.py` both ways.
- `scripts/test_sign_release.py`: the command-line tool.

### Render test

The drawing code needs LovyanGFX, so it has its own test: `scripts/run_render_test.sh` builds the radar's real drawing code (`radar_display.cpp`, `runway_overlay.cpp`, `strip_canvas.cpp`) against the real LovyanGFX library, compiled for the host so that it draws into memory, and renders a 240×240 screen without an ESP32.

```bash
scripts/run_render_test.sh           # smooth-font, bitmap-font and sanitizer builds
scripts/run_render_test.sh --quick   # smooth-font build only
```

It needs `g++` only. LovyanGFX is taken from `$LOVYANGFX_DIR` (its `src/` folder), from PlatformIO's `.pio/libdeps`, or cloned at the pinned tag (`LOVYANGFX_TAG` in the script: keep it in step with `lib_deps` in `platformio.ini`); the compiled library is kept in `.pio/render-test/`. The project's files are built with the firmware's warning flags and `-Werror`. What it checks (`test/render/test_radar_render.cpp`):

- 240 scenes (12 radar positions — open country, high latitude, southern hemisphere, the date line, 8 real airports — × the 4 range presets × empty sky, a few planes, a busy sky, the 64-aircraft cap, planes at the rim, an aircraft on top of the radar twice; km or miles, runways on or off, stale or fresh data) give **exactly the same pixels at every strip height**: all 20 divisors of 240 for the first 24 scenes, 8 heights for the rest. A strip of 240 rows is the whole-frame case, the reference;
- a refresh after new aircraft data draws the same frame as a full redraw;
- every pixel of the screen is written by every frame (two different fill colours must both end up fully covered), so no area is ever left unpainted or stale;
- the last-resort path that draws straight on the panel gives the same picture as the strips, and the stats shown in the serial log are right;
- `StripCanvas::drawWideLine` matches the library's own `drawWideLine` where the library clips correctly.

To prove that a change to the drawing code left the picture untouched, compile the test against the old sources with `-DRENDER_REFERENCE` (it then only renders), run it with `--write-hashes FILE`, then run the new build with `--check-hashes FILE` (see the header of the test). `--dump-ppm FILE SCENE` writes one scene as an image.

## Improvements considered but not made

- **Moving the runway dataset (`large_airports_data.cpp`, ~2,800 lines) out of flash and into LittleFS/SPIFFS as a binary blob.** Flash is the tightest resource: the firmware fills about 81% of a 1.5 MB app slot with the two-slot OTA partition table (see `partitions/plane_radar.csv`). It wasn't done because it trades a compiled, always-valid C array for runtime file I/O with new failure modes (mount failures, a missing or corrupt filesystem image) that can't be tested without real hardware — exactly the kind of change that should be made and verified on the device, not shipped speculatively. If flash pressure becomes a real constraint, this is the first place to look.
- **Certificate pinning for the ADS-B TLS connection.** See **Security notes** above — the fragility trade-off (breaking on every cert rotation, on a device with no other way to fetch a fresh cert) didn't seem worth it for read-only public data, but the hook (`setCACert`) is a one-line change if your threat model differs.
- **A second ADS-B data source as automatic fallback** (e.g. adsb.lol or airplanes.live, both drop-in-compatible with the same query shape). Worth doing if adsb.fi's public endpoint ever becomes unreliable, but it doubles the HTTP/JSON-handling code path for a dependency that has been stable so far — not worth the complexity pre-emptively.

## Configuration

Edit **`include/config.h`** for hardware and behavior:

| Area | Keys / notes |
|------|----------------|
| Version | `FIRMWARE_VERSION` — printed at boot and sent in the API's User-Agent; set at build time with `FIRMWARE_VERSION=1.2.3 pio run -e supermini` (the release workflow takes it from the git tag) |
| Portal | `kPortalApName`, `kPortalIp`, `kPortalHostname` / `kPortalHostUrl` (mDNS; needs `-DWM_MDNS` in `platformio.ini`) |
| Wi‑Fi timing | `kWifiConnectAttemptMs`, `kWifiConnectAttempts`, `kWifiDownGraceMs`, `kWifiReconnectIntervalMs`, portal timeouts `kWifiPortalTimeoutSec` (no network saved; `0` = wait for the user) and `kWifiPortalTimeoutSavedSec` (a saved network is unreachable; default 120 s) |
| BOOT | `kBootPin`, `kBootResetHoldMs`, `kBootTapMinMs`, `kBootBounceMs` |
| Display SPI | pins, `kDisplayInvert`, `kDisplayRgbOrder`, `kDisplaySpiWriteHz` |
| Default location | `kDefaultRadarLat`, `kDefaultRadarLon` (until portal overrides) |
| ADS-B | `kAdsbFetchIntervalMs`, `kAdsbMinRequestSpacingMs`, `kAdsbMaxBackoffMs`, `kAdsbRateLimitedDefaultMs`, `kAdsbRetryAfterMaxMs`, `kAdsbQueryStepDeg` (`0` = send exact coordinates), `kAdsbShowGroundAircraft`, `kAdsbStaleMs` |
| Self-healing | `kAdsbStallRestartMs`, `kAdsbMinHeapBlockBytes`, `kLowHeapRestartSkips`, `kHealthLogIntervalMs` |
| Altitude tag | `kAltitudeFeetToUnit`, `kAltitudeUnitSuffix`, `kAltitudeUnitDecimals` (default: km) |
| Watchdog | `kWatchdogTimeoutSec` |
| Firmware updates | `UPDATE_GITHUB_REPO`, `UPDATE_PUBLIC_KEY` (build flags or environment variables at build time; see **Firmware updates**), `kBoardId`, `kUpdateConfirmAfterMs`, `kUpdateTrialMs` |
| Address screen | `kAddressScreenMs`: how long the radar shows its address (`plane-radar.local` and its IP) after it first connects at power-on |
| ArduinoOTA | `OTA_PASSWORD` (a build flag; off without one), `kOtaPort` |

Range presets: `include/ui/radar_range.h` (`kRangePresets`).

## Project layout

```
include/
  config.h
  hardware/
    boot_button.h          — BOOT button: interrupt edge queue + polling -> ButtonTracker
    lgfx_config.hpp
    display.h
    display_font.h
    watchdog.h
  data/
    large_airports.h
  ui/
    radar_theme.h
    radar_range.h
    radar_projection.h     — latitude/longitude -> pixels for one frame
    radar_display.h        — frame buffer set-up (call at boot) and screen drawing
    strip_canvas.h         — one horizontal strip of the screen: whole-screen coordinates, clipped
    runway_overlay.h
    status_screens.h
  services/
    wifi_setup.h
    radar_location.h
    adsb_client.h
    adsb_task.h            — runs the ADS-B fetch on its own FreeRTOS task
    health.h               — boot reason, health log line, self-healing restarts
    fw_update.h            — firmware updates from GitHub Releases, when the owner asks
    ota.h                  — ArduinoOTA for the developer's bench (off without a password)
  util/                  — pure, Arduino-free logic (see Testing)
    format.h
    fetch_schedule.h       — when the next ADS-B request may start (cadence, backoff)
    buffered_stream_reader.h — network body -> ArduinoJson, with timeouts
    adsb_parse.h           — one adsb.fi JSON object -> Aircraft record
    adsb_stream.h          — streaming parse, one aircraft at a time, nearest 64 kept
    api_query.h            — rounded query position and padded radius
    retry_after.h          — Retry-After header
    request_guard.h        — which setup-portal requests to trust
    button_tracker.h       — debounce, tap vs. hold
    geo.h                  — flat-earth frame, date line, integer pre-filter
    disc_clip.h            — cut a line off at the radar disc
    fw_release.h           — release statement, versions, URLs, image checks
    fw_manifest.h          — manifest.json
    fw_crypto.h            — SHA-256 and ECDSA P-256 signature check (mbedTLS)
data/
  ui_font.vlw              — embedded smooth UI font (Noto Sans Bold)
partitions/
  plane_radar.csv          — two app slots for updates
tools/
  release-signer.html      — creates the signing key and signs releases, offline in a browser
scripts/
  build_large_airports.py  — regenerates the runway data
  test_build_large_airports.py
  run_host_tests.sh        — builds and runs every host-side test
  run_render_test.sh       — the radar's drawing code on the real LovyanGFX, on the host
  run_fw_update_tests.sh   — the update service with fakes (part of run_host_tests.sh)
  run_web_tests.sh         — the update page and the release tools in a real browser
  lib_mbedtls.sh           — builds mbedTLS (the ESP32 core's version) for host tests
  sign_release.py          — the release signer on the command line and in CI
  test_sign_release.py
  firmware_version.py      — stamps FIRMWARE_VERSION, UPDATE_GITHUB_REPO, UPDATE_PUBLIC_KEY (pre-script)
  merge_firmware.py        — builds the single web-flashable image (PlatformIO post-script)
  merge-firmware.sh
src/
  main.cpp
  data/
    large_airports_data.cpp
  hardware/
  ui/
  services/
  util/
test/
  native/                — host-side unit tests, no ESP32 toolchain needed
    test_format.cpp
    test_fetch_schedule.cpp
    test_adsb_parse.cpp
    test_http_helpers.cpp
    test_request_guard.cpp
    test_button_tracker.cpp
    test_geo.cpp
    test_disc_clip.cpp
    test_projection.cpp
    test_runway_scan.cpp
    test_fw_release.cpp
  fw_update/             — the update service with fakes (scripts/run_fw_update_tests.sh)
    test_fw_update.cpp
    stubs/               — network, GitHub, flash writer, OTA state, web server, clock
  web/                   — browser tests (scripts/run_web_tests.sh)
    mock_radar.py          — stand-in for the radar's update interface
    test_update_page.py
    test_release_signer.py
    verify_release.cpp     — the radar's release checks as a host program
  render/                — the drawing code against LovyanGFX on the host (scripts/run_render_test.sh)
    test_radar_render.cpp
    stubs/               — Arduino, Preferences, GPIO and display stand-ins
    vlw_embed.S          — embeds data/ui_font.vlw (vlw_none.S: bitmap-font flavour)
```

## Wiring (GC9A01 ↔ ESP32-C3 Super Mini)

| Display | ESP32-C3 |
|---------|----------|
| VCC | 3V3 |
| GND | GND |
| RST | GPIO **0** |
| CS | GPIO **1** |
| DC | GPIO **10** |
| SDA (MOSI) | GPIO **3** |
| SCL (SCLK) | GPIO **4** |
| BOOT (user) | GPIO **9** |

## Build

```bash
pio run -t upload
pio device monitor
```

- PlatformIO env: **`supermini`**
- Serial: **115200** baud
- USB CDC on boot enabled in `platformio.ini` for the Super Mini
- Built as C++17. The Arduino-ESP32 platform appends `-std=gnu++11` after `build_flags`, so `platformio.ini` removes it with `build_unflags` for the standard it asks for to take effect.
- The project's own code is compiled with `-Wall -Wextra -Wshadow -Wundef -Wformat=2 -Wnull-dereference` (`build_src_flags`, so the libraries' warnings stay out of it). It builds without a single warning and CI fails if one appears.
- The version printed at boot and sent in the User-Agent is `FIRMWARE_VERSION` in `config.h`; `FIRMWARE_VERSION=1.2.3 pio run -e supermini` overrides it for one build.

### Web-flashable release image

Single `.bin` for [esptool-js](https://espressif.github.io/esptool-js/) and similar tools (ESP32-C3, 4 MB, flash at **0x0**):

```bash
chmod +x scripts/merge-firmware.sh   # once
./scripts/merge-firmware.sh
```

Writes `release/plane-radar-merged.bin`. Skip rebuild if firmware is already built:

```bash
./scripts/merge-firmware.sh --no-build
```

Or via PlatformIO only (output: `.pio/build/supermini/firmware-merged.bin`):

```bash
pio run -e supermini
pio run -t merge -e supermini
```

Put the board in download mode (hold **BOOT**, tap **RESET**), then flash with Chrome/Edge over USB.

A first flash always needs USB (see above); after that, radars update themselves from GitHub when their owner asks — see **Firmware updates**. The partition table (`partitions/plane_radar.csv`) has two 1.5 MB app slots (`ota_0`/`ota_1`) to support this; the merged `.bin` offsets above are unchanged.

### CI and releases (GitHub Actions)

| Workflow | When | What it does |
|----------|------|--------------|
| [Build](.github/workflows/build.yml) | Push / PR to `main` | `native-tests`: the host-side tests (64-bit, 32-bit, sanitizers; the update service with its fakes), the render test and the web tests. `firmware`: the real build with PlatformIO pinned to 6.2.0, failing on any warning from `src/` or `include/`, then uploads artifact `plane-radar-supermini` (merged + split `.bin` files, ~90 days) |
| [Release](.github/workflows/release.yml) | Git tag `v*` (e.g. `v1.3.1`) | Runs the host-side tests first (a firmware whose tests fail is never published), builds with `FIRMWARE_VERSION` taken from the tag (`v1.3.1` → `1.3.1`), signs it when the signing key is set up (below), and attaches to the GitHub Release: `plane-radar-v1.3.1.bin` + `.sha256` (full-flash image, for USB), and `manifest.json` + `plane-radar-update.bin` (what radars install) |

The Build workflow has read-only repository permissions; the Release workflow gets write access only to attach the file to the release.

To ship a version users can download:

```bash
git tag v1.0.0
git push origin v1.0.0
```

The release workflow builds firmware in CI and attaches the merged image to the release. Download from **Releases** on GitHub, then flash at **0x0** (ESP32-C3, 4 MB).

For radars to see a CI release as an update, give the workflow your key, in the repository's **Settings → Secrets and variables → Actions**:

- secret **`PLANE_RADAR_SIGNING_KEY`**: the whole contents of `plane-radar-signing-key.json`;
- variables **`PLANE_RADAR_UPDATE_REPO`** (`owner/name` of the releases repository) and **`PLANE_RADAR_PUBLIC_KEY`** (the public key), unless they are already in `include/config.h`.

Without the secret, releases are built but not signed (the workflow says so), and radars do not offer them. A tag with a `-` in it (`v1.4.0-beta1`) is published as a **pre-release**, for USB only and unsigned: radars only look at the latest full release, so a test build never hides the signed one. A version with leading zeros (`v1.03.1`) does not build at all (`scripts/firmware_version.py`): radars fetch the image from the release tagged exactly `v` + the plain version.

## Dependencies

Tested with, and held to patch updates of (`~` ranges in `platformio.ini`, so a new minor release can't change behavior unnoticed):

- [LovyanGFX](https://github.com/lovyan03/LovyanGFX) 1.2.29
- [WiFiManager](https://github.com/tzapu/WiFiManager) 2.0.17
- [ArduinoJson](https://github.com/bblanchon/ArduinoJson) 7.4.3
- PlatformIO platform `espressif32` 6.5.0 (Arduino-ESP32 2.0.14)
- ArduinoOTA (bundled with the ESP32 Arduino core — no extra `lib_deps` entry needed)

Moving to a newer minor version is a deliberate step: change the range, run `scripts/run_host_tests.sh`, `scripts/run_render_test.sh` and a firmware build, and update `ARDUINOJSON_TAG` in `scripts/run_host_tests.sh` if ArduinoJson changes, or `LOVYANGFX_TAG` in `scripts/run_render_test.sh` if LovyanGFX does (the render test relies on how the library clips and draws; a new version could change a pixel).
