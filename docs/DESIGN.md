# Moon KDS — Design Document

| | |
|---|---|
| **Status** | DRAFT, revision 0.5 |
| **Owner** | Moonwake Roasters |
| **Last updated** | 2026-11-01 |
| **Benchmark** | Fresh KDS (the feature set and reliability we must meet or beat) |

> This document is the single source of truth for requirements, architecture and decisions.
> Every requirement has a stable ID (e.g. `FR-ORD-01`). Code, tests and commits should reference those IDs.
> Items marked **❓** are open questions to settle before coding starts.

---

## 1. Purpose & scope

Build a kitchen display system (KDS) from scratch for a coffee shop that takes orders on a
**Square Terminal**. Orders go to **several Android tablets**, each showing a different slice of the
work (e.g. espresso bar, cold bar, food, expo/pickup). An action on one tablet, such as bumping
an order, must show up on every other tablet straight away.

A **cloud component** records statistics (ticket times, throughput, item mix), holds configuration
and backups, and gives devices a second way to talk to each other.

**The top priority is robustness.** The shop must keep running when the internet goes down, and
a single tablet failing must not stop it.

### 1.1 Goals
1. Feature parity with Fresh KDS for what a coffee shop actually uses (see §4).
2. Keep working through internet outages, Wi-Fi blips, and a tablet dying or rebooting.
3. Fast and easy to read during a rush: one tap to bump, readable from about 1.5 m away, no lag.
4. Accurate speed-of-service statistics.
5. Cheap to run: off-the-shelf Android tablets, low cloud cost, no per-device subscription.

### 1.2 Non-goals (for v1)
- Replacing Square POS or handling payments.
- Inventory management, labour scheduling, payroll.
- Running more than one location (the design should leave room for it, but v1 won't build it).
- iOS tablets.

---

## 2. Background: how Fresh KDS works (benchmark reference)

### 2.1 Two integration modes
Fresh KDS (Android/iPad app) can get orders from Square in two ways, chosen per tablet in
*Settings › General › Integration Type*:

| | **Printer mode** ("printer emulation") | **Orders API mode** |
|---|---|---|
| Transport | **Local Wi-Fi/LAN**: the Square device "prints" to the tablet. **Not Bluetooth.** | Square cloud → Fresh cloud → tablet |
| Needs internet? | No: Fresh's docs say it works over the local network without internet | Yes |
| Square-side setup | Each tablet added as a **printer station** (*Settings › Hardware › Printers › Create Printer Station*, model **"Fresh KDS"**), with categories picked per station. Repeat on every Square device. | One OAuth connection in the Fresh dashboard |
| Routing | Done **in Square** by category per printer station | Done in Fresh |
| Data quality | "Occasional parsing issues" | Structured data |
| Lost tickets | **"Cannot be recovered"** after network problems (fire-and-forget) | Recoverable |
| Fresh's recommendation | Use only for locations with poor or unreliable internet | **Preferred** |

Operational quirks of printer mode, from Fresh's troubleshooting docs:
- The tablet only advertises itself as a printer **while the app is open in the foreground**. Sleep or minimising breaks it.
- Android's **randomised (private) MAC address must be turned off** (*Wi-Fi › Advanced › MAC address type › Device MAC*), and a **static IP / DHCP reservation** is recommended. Square can also add the printer by IP (*Advanced Printer Setup*).
- It doesn't work if the Square device is acting as a hotspot. Mixed 2.4/5 GHz SSIDs cause trouble.
- **"Fresh KDS" appears as its own printer model inside Square POS.** A Square developer-forum answer says
  "Fresh KDS read like a printer would", that "they don't actually get the data from APIs", and that the
  implementation is undocumented.
- **Origins:** Fresh started out as a pure printer emulator. A Loyverse forum user reports "FreshKDS already
  integrated an Epson Printer Emulator when you select Square as your POS". Early on, Fresh only worked with
  the iOS Square app, which is consistent with emulating a printer that app supported. The dedicated "Fresh KDS"
  entry came later.
- **No evidence was found of a handshake, licence check or authentication** in the Fresh printer entry. The most
  likely explanation is a Square printer driver that sends a mildly altered (likely text-based) print stream. This
  is unverified until we capture it (spike S1).

Sources: [Fresh: treat KDS as a printer](https://help.freshkds.com/en/articles/6289591-how-do-i-treat-fresh-kds-as-a-printer-square),
[Fresh: printer emulation troubleshooting](https://help.freshkds.com/en/articles/6411775-why-won-t-my-square-pos-connect-to-fresh-kds-printer-emulation),
[Fresh: Orders API](https://help.freshkds.com/en/articles/5983132-how-do-i-use-fresh-kds-with-orders-api-square),
[Square dev forum: custom KDS as hardware](https://developer.squareup.com/forums/t/how-can-i-connect-my-custom-kds-web-app-as-hardware-like-fresh-kds/23570).

### 2.2 What this means for us
Square POS only prints to printers it recognises: its supported **Star Micronics** and **Epson** network
models, plus partner models like Fresh. To be a "printer" without Fresh's special slot, our tablet would have to
**emulate a supported Star or Epson LAN printer**: answer the vendor's discovery protocol (Star: UDP 22222;
Epson: its own discovery) and accept print jobs on TCP 9100.

⚠ **Key risk:** independent reverse-engineering reports say **Square sends tickets to Star and Epson printers as
raster bitmaps, not text** ([Epson TM-m30 analysis](https://wes4m.io/posts/epson_rev/);
[Star TSP143 bridge](https://github.com/jeff-johnson/tsp143-bridge)). If that's true, an emulated printer
receives a **picture of the ticket**, and we would have to **OCR** it. The "Fresh KDS" model slot and impact-printer models are the most
likely to get text instead (see S1); otherwise we to get items and modifiers. OCR of
machine-rendered, fixed-font text is very accurate, but it's a new failure surface (layout changes in
Square updates, long modifier lists, special characters). This must be proven in the **M0 spike** before we commit.

⚠ **Conflicting evidence about offline:** Fresh says printer mode works without internet. Some Square
community and accessory pages say network printers **can't be used in Square's Offline Mode**, and a
community answer says offline Terminal sales don't reach Fresh. → **Test on the actual Terminal with the WAN unplugged** (M0).

### 2.3 Features to match
Routing by category/item to screens, expo screen, item-level and order-level bumping, recall,
colour-coded timers, all-day item counts, modifier highlighting, bump bar support, customer SMS,
customer-facing status screen, prep-time reports in a web dashboard, per-screen layouts, and an
option to show open (unpaid) checks.

**Where we aim to beat it:**
1. **One Square-side printer, not eight.** Square sends every ticket to one or two "ingest" tablets, and our own
   inter-tablet sync does the per-station routing (see §7.3). This removes most of the per-tablet Square setup.
2. **Printer path and API path at the same time**, reconciled, so we get the LAN/offline speed of printer mode
   **and** the structured, recoverable data of the API. Fresh makes you pick one.
3. Peer-to-peer replication, so no single tablet or cloud is a point of failure.

❓ **Q-BENCH-1:** Is Fresh currently running in the shop, and in which mode? Which features do you rely on
and which annoy you?

---

## 3. Shop context & assumptions

| ID | Assumption | Confirm? |
|---|---|---|
| A-01 | **Exactly 1 Square Terminal** running Square Point of Sale | ✔ confirmed |
| A-02 | **8 Android tablets today; the count will change.** The design must handle adding or removing tablets without Square-side changes (FR-RTE-07) | ✔ confirmed |
| A-03 | One shop Wi-Fi network (one router or AP), with clients allowed to see each other (no "AP isolation") | ❓ |
| A-04 | Peak volume of about 60–120 orders per hour, 1–8 items per order | ❓ |
| A-05 | Order sources: in-person on the Terminal, plus Square Online and online ordering/delivery integrations that land in Square Orders | ❓ |
| A-06 | Stations: Espresso/Hot bar, Cold bar, Food/Pastry, Expo/Handoff | ❓ |
| A-07 | Staff are not technical, so setup and recovery must be guided and run from the tablets | ✔ assumed |

---

## 4. Functional requirements

Priority: **P0** = must have for first use in the shop, **P1** = needed for parity with Fresh, **P2** = nice to have.
Fresh column: ✔ = Fresh has it, ~ = partial, ✘ = Fresh doesn't have it, ? = unknown.

### 4.1 Order ingestion (ORD)
| ID | Requirement | Pri | Fresh | Status |
|---|---|---|---|---|
| FR-ORD-01 | Receive every Square order (Terminal, Online, third-party) with line items, modifiers, notes, customer name, fulfillment type and source | P0 | ✔ | Draft |
| FR-ORD-02 | A new order shows on screen within **3 s (p95)** of payment while online | P0 | ✔ | Draft |
| FR-ORD-03 | Ingestion is **idempotent**: duplicate webhooks or polls never create duplicate tickets | P0 | ✔ | Draft |
| FR-ORD-04 | Reconciliation: periodic poll of Square Orders API catches anything a webhook missed | P0 | ? | Draft |
| FR-ORD-05 | Handle order updates, cancellations and refunds (ticket marked or removed, with a visual alert) | P0 | ✔ | Draft |
| FR-ORD-06 | Scheduled or future orders (Square Online pickup times) appear at a configurable lead time before they're due | P1 | ✔ | Draft |
| FR-ORD-07 | Manual "offline order" entry on any tablet when Square is unreachable (see §6.1) | P1 | ✘ | ❓ |
| FR-ORD-08 | Orders taken on the Terminal in Square's offline mode are ingested once Square uploads them, flagged "late" and not double-counted | P0 | ? | Draft |
| FR-ORD-09 | **Primary ingest = LAN printer path:** a tablet appears to the Terminal as a network kitchen printer and turns each ticket into a structured order (parse text, or OCR if raster) | P0 | ✔ | Spike (M0) |
| FR-ORD-10 | **Printer path redundancy:** two or more tablets can be ingest printers at once, with duplicate tickets merged into one order | P0 | ✘ | Draft |
| FR-ORD-11 | **Reconciliation:** when the API path is available, each printer-path ticket is matched to its Square order (by ticket/order number and time). The API data replaces any OCR guesses, and missing tickets are back-filled | P0 | ✘ | Draft |
| FR-ORD-12 | **Parse confidence:** low-confidence parses are shown as is, with a "check ticket" marker, never silently dropped. The raw ticket image is kept so staff can view it | P0 | ✘ | Draft |
| FR-ORD-13 | The ingest tablet keeps its printer listener running in a foreground service, so it isn't lost when the screen sleeps or the app is in the background (Fresh's weak spot) | P0 | ✘ | Draft |

### 4.2 Routing & stations (RTE)
| ID | Requirement | Pri | Fresh | Status |
|---|---|---|---|---|
| FR-RTE-01 | Route line items to stations by Square category, item, or modifier rules | P0 | ✔ | Draft |
| FR-RTE-02 | One order can split across several stations, each seeing only its own items (with an optional "other items" hint) | P0 | ✔ | Draft |
| FR-RTE-03 | Expo station sees the whole order and the bump status from every prep station | P0 | ✔ | Draft |
| FR-RTE-04 | Any tablet can take over any station's role (failover) in two taps or fewer | P0 | ~ | Draft |
| FR-RTE-05 | A tablet can show more than one station (e.g. a quiet day runs on one screen) | P1 | ✔ | Draft |
| FR-RTE-06 | Routing rules are edited on a tablet or the cloud dashboard and synced everywhere | P1 | ✔ | Draft |
| FR-RTE-07 | **Routing lives in Moon KDS, not Square.** Adding, removing or reassigning a tablet needs no change in Square POS settings | P0 | ✘ | Draft |
| FR-RTE-08 | Optional "Square-routed" mode: if a tablet is added as its own printer station in Square, it shows exactly what Square sends (Fresh-style), and those tickets still sync for expo and stats | P2 | ✔ | ❓ |

### 4.3 Display & interaction (UI)
| ID | Requirement | Pri | Fresh | Status |
|---|---|---|---|---|
| FR-UI-01 | Ticket grid (tiles) layout, plus an optional rail/list layout | P0 | ✔ | Draft |
| FR-UI-02 | Elapsed-time timer per ticket with colour thresholds (e.g. green, then yellow at 3 min, red at 5 min), configurable per station | P0 | ✔ | Draft |
| FR-UI-03 | Bump a whole order with one tap; bump single items by tapping an item | P0 | ✔ | Draft |
| FR-UI-04 | Recall / undo the last N bumped orders (at least the last 20, or last 30 min) | P0 | ✔ | Draft |
| FR-UI-05 | Highlight modifiers, allergies and notes (e.g. "OAT", "EXTRA HOT", allergy keywords in red), driven by the formatting rules in §4.8 | P0 | ✔ | Draft |
| FR-UI-06 | Show order number or customer name, source (in-store, online, delivery) and dine-in or to-go | P0 | ✔ | Draft |
| FR-UI-07 | All-day counts: running totals of items outstanding (e.g. "6× Oat Latte") | P1 | ✔ | Draft |
| FR-UI-08 | Audible chime on a new order, configurable per station | P0 | ✔ | Draft |
| FR-UI-09 | Item consolidation: identical items with identical modifiers grouped as "2×" | P1 | ✔ | Draft |
| FR-UI-10 | Mark in progress / started (e.g. tap the header), visible to other stations | P1 | ~ | Draft |
| FR-UI-11 | Prioritise or rush a ticket; pin it to the front | P2 | ? | Draft |
| FR-UI-12 | Hardware bump bar / keypad / arcade-button support (see §4.9) | P0 | ✔ | Draft |
| FR-UI-13 | Status bar on every screen showing connectivity (LAN peers, cloud, Square) at all times | P0 | ~ | Draft |
| FR-UI-14 | Dark theme by default, large touch targets (≥ 48 dp, ideally 64 dp), readable from 1.5 m | P0 | ✔ | Draft |

### 4.4 Cross-device sync (SYN)
| ID | Requirement | Pri | Fresh | Status |
|---|---|---|---|---|
| FR-SYN-01 | A bump, recall or start on any tablet shows on all tablets within **500 ms (p95)** over LAN | P0 | ~ | Draft |
| FR-SYN-02 | Sync works over the LAN **without internet** | P0 | ✘? | Draft |
| FR-SYN-03 | Sync falls back to the cloud relay when the LAN path fails (e.g. a tablet on a different SSID) | P1 | ✔ | Draft |
| FR-SYN-04 | A tablet that reboots or rejoins catches up to the full current state automatically, with no operator action | P0 | ✔ | Draft |
| FR-SYN-05 | Conflicting actions (two tablets bump or recall the same ticket at once) resolve deterministically, and every tablet ends up in the same state | P0 | ? | Draft |
| FR-SYN-06 | Devices discover each other automatically on the LAN (mDNS), with manual IP entry as a fallback | P0 | ~ | Draft |

### 4.5 Customer-facing (CUS)
| ID | Requirement | Pri | Fresh | Status |
|---|---|---|---|---|
| FR-CUS-01 | Customer order-status board ("Preparing / Ready") on a spare tablet or TV | P1 | ✔ | ❓ |
| FR-CUS-02 | SMS "your order is ready" when Expo bumps, if a phone number is on the order | P2 | ✔ | ❓ |

### 4.6 Statistics & cloud (STA)
| ID | Requirement | Pri | Fresh | Status |
|---|---|---|---|---|
| FR-STA-01 | Record a timestamp for every lifecycle event (received, displayed, started, item bumped, station bumped, expo bumped, recalled) | P0 | ✔ | Draft |
| FR-STA-02 | Events are kept on the devices while offline and uploaded once connectivity returns, **with no loss** | P0 | ? | Draft |
| FR-STA-03 | Web dashboard: average and p90 ticket time by station, hour and day; orders per hour; item mix; slowest items | P1 | ✔ | Draft |
| FR-STA-04 | Export to CSV | P2 | ✔ | Draft |
| FR-STA-05 | Live view of current queue depth and ticket ages from anywhere (owner on phone) | P2 | ? | Draft |

### 4.7 Configuration & admin (ADM)
| ID | Requirement | Pri | Fresh | Status |
|---|---|---|---|---|
| FR-ADM-01 | Pair a new tablet by scanning a QR code or entering a PIN shown on an existing tablet or the dashboard | P0 | ~ | Draft |
| FR-ADM-02 | Admin settings behind a PIN | P0 | ✔ | Draft |
| FR-ADM-03 | Config is versioned and synced (local and cloud), so a replacement tablet comes up fully configured | P0 | ✔ | Draft |
| FR-ADM-04 | Kiosk mode: app starts on boot, staff can't leave it, and it stays awake | P0 | ~ | Draft |
| FR-ADM-05 | Square OAuth connection is managed from the dashboard | P0 | ✔ | Draft |
| FR-ADM-06 | Guided setup checklist for ingest tablets: Device MAC, static IP / DHCP reservation, battery-optimisation exemption, Square printer station test print | P0 | ~ | Draft |

### 4.8 Custom ticket formatting (FMT)
Staff need to spot certain options at a glance, e.g. **decaf** or **dark roast** espresso, or **oat** vs
**almond** milk. Formatting is configured on a settings screen (and on the dashboard) as a list of **rules**.

| ID | Requirement | Pri | Fresh | Status |
|---|---|---|---|---|
| FR-FMT-01 | **Settings screen for formatting rules** (behind the admin PIN): create, edit, reorder, enable/disable and delete rules | P0 | ~ | Draft |
| FR-FMT-02 | **Rule match** on any of: modifier (e.g. "Decaf", "Oat Milk"), modifier group (e.g. "Milk"), item, category, keyword in notes, order type/source. Match by Square catalog ID when known (API path), **and** by normalised name (printer path), with case- and whitespace-insensitive matching and an optional regex | P0 | ✘ | Draft |
| FR-FMT-03 | **Rule styles:** text colour, background/pill colour, bold, uppercase, larger font, icon/badge (e.g. ☕ DECAF), border or tint on the whole item or the whole ticket, optional blink for critical rules (e.g. allergies) | P0 | ~ | Draft |
| FR-FMT-04 | **Rename / abbreviate / hide:** display "Oat Milk" as "OAT", hide noise modifiers such as "Regular" or "Whole Milk (default)", and set the modifier order (e.g. shots → milk → syrup → temp) | P0 | ~ | Draft |
| FR-FMT-05 | **Scope:** each rule applies to all stations or to selected stations or tablets (e.g. milk colours only on the bar) | P0 | ✘ | Draft |
| FR-FMT-06 | **Conflict resolution:** rules have an explicit priority. When several match, styles merge in priority order (highest wins per property) | P0 | ✘ | Draft |
| FR-FMT-07 | **Live preview** in the editor with a sample ticket and real recent tickets | P0 | ✘ | Draft |
| FR-FMT-08 | **Option picker** built from the Square catalog (API path) and from modifier names seen on recent tickets, so staff pick "Almond Milk" from a list instead of typing it | P1 | ✘ | Draft |
| FR-FMT-09 | **Accessibility:** colour is never the only signal. Every colour rule also gets text or a badge; the palette is colour-blind safe and legible on the dark theme (≥ 4.5:1 contrast check in the editor) | P0 | ✘ | Draft |
| FR-FMT-10 | **Rules sync** to all tablets as versioned `ConfigChanged` events, work offline, and can be exported/imported as JSON (back up and share between shops) | P0 | ✘ | Draft |
| FR-FMT-11 | **Item-level styles:** e.g. tint all iced drinks blue, or flag items with 3+ modifiers as "complex" | P1 | ✘ | Draft |
| FR-FMT-12 | **Day-one presets (decided, D-009): milk types and espresso types only.** Each milk option gets its own colour (e.g. oat, almond, soy, coconut, skim, half & half). Each espresso option gets its own colour (e.g. decaf, half-caf, dark roast, other house espresso options). The exact list comes from the shop's Square modifier groups. Other presets (extra shot, iced, allergy keywords) ship disabled | P0 | ✘ | Decided |
| FR-FMT-13 | **Sound or attention rules:** a matching item can trigger a distinct chime (e.g. allergy) | P2 | ✘ | Draft |

Example rule set (starter preset sketch):

| Match | Display as | Style |
|---|---|---|
| Modifier "Decaf" | `DECAF` | purple pill, white bold text |
| Modifier "Half Caf" | `½ CAF` | purple outline pill |
| Modifier "Dark Roast" (espresso group) | `DARK` | dark-brown pill, cream text |
| Modifier "Oat Milk" | `OAT` | amber pill |
| Modifier "Almond Milk" | `ALMOND` | tan pill with ◆ badge |
| Modifier "Soy Milk" | `SOY` | green pill |
| Modifier "Whole Milk" | *(hidden, default)* | — |
| Notes contain "allerg", "nut", "gluten" | as typed | red tint on the whole ticket + ⚠ badge, blink 3 s |

### 4.9 Peripheral input: hotkeys, bump bars and arcade buttons (INP)
Goal: staff can bump items and orders with **big physical buttons** (arcade-style, wet-hands friendly)
without touching the screen. Touch always keeps working alongside.

| ID | Requirement | Pri | Fresh | Status |
|---|---|---|---|---|
| FR-INP-01 | Accept input from **USB HID keyboards** (wired, via USB-C OTG) and **Bluetooth HID keyboards** | P0 | ✔ | Draft |
| FR-INP-02 | Accept input from **HID gamepad/joystick encoders** (the cheap "zero-delay" USB arcade encoders appear as gamepads, not keyboards) | P0 | ✘ | Draft |
| FR-INP-03 | **Per-tablet key mapping screen** with a "press a button to assign" learn mode. Any key or gamepad button can map to any action | P0 | ~ | Draft |
| FR-INP-04 | **Actions:** next/previous ticket, next/previous item, bump selected item, bump selected ticket, **direct bump ticket by position (1–9)**, start/in-progress, recall last, undo, scroll page, toggle all-day view, rush/pin | P0 | ✔ | Draft |
| FR-INP-05 | **Visible selection cursor** (thick, high-contrast outline) that the buttons move. The cursor follows sensibly when tickets arrive or leave (never jumps onto a newly arrived ticket mid-action) | P0 | ✔ | Draft |
| FR-INP-06 | **Long-press and double-press** can be mapped separately (e.g. long-press "bump" = recall), with configurable thresholds | P1 | ✘ | Draft |
| FR-INP-07 | **Safety:** debounce (default 150 ms), ignore repeats while held, and a short undo toast after every hardware bump | P0 | ~ | Draft |
| FR-INP-08 | **Presets** for common layouts: 2-button (next + bump), 4-button (prev/next/bump/recall), 10-key bump bar (Fresh-style), numeric keypad | P1 | ~ | Draft |
| FR-INP-09 | Device status shown in the status bar (keyboard/gamepad connected), with an alert if a configured device disconnects | P1 | ✘ | Draft |
| FR-INP-10 | Input never leaks out of kiosk mode (no system shortcuts like Home/Alt-Tab reaching Android) | P0 | ✘ | Draft |
| FR-INP-11 | **Remote buttons:** a button box can act on any station's tickets (action sent as an event over sync), not just the tablet it's paired with | P1 | ✘ | Draft |
| FR-INP-12 | **Initial scope (D-010): only the Expo tablet gets physical buttons.** Other stations stay touch-only, but the same input stack is available to them later | P0 | — | Decided |
| FR-INP-13 | **End-to-end acknowledgement:** each press shows a result on the button box itself (LED/beeper: green = applied, red = rejected/failed) **only after** the KDS has applied the action, plus on-screen feedback. Staff never have to guess whether a press counted | P0 | ✘ | Draft |
| FR-INP-14 | **Exactly-once presses:** each press carries a device ID and sequence number. Retransmits are deduped, and nothing is replayed after a reconnect unless it's still valid (a press older than 2 s is dropped and shown red) | P0 | ✘ | Draft |
| FR-INP-15 | **Link health:** heartbeat every ≤ 1 s. A lost link is detected within 3 s and shows a loud banner on Expo ("BUTTONS DISCONNECTED — use touch") and a red LED on the box | P0 | ✘ | Draft |
| FR-INP-16 | **Reliability targets (verified in spike S6 at the real bar location):** 0 missed and 0 duplicate presses in 1,000; p95 press-to-screen latency ≤ 150 ms; **no lost first press after idle** (the main Bluetooth weakness) | P0 | ✘ | Draft |
| FR-INP-17 | **Buttons follow the Expo role:** if the Expo tablet fails and another tablet takes the role (FR-RTE-04), the button box rebinds automatically without re-pairing | P1 | ✘ | Draft |
| FR-INP-18 | **Power:** prefer mains (USB wall adapter) at the box. If battery powered, report the battery level and alert at 20% | P0 | ✘ | Draft |
| FR-INP-19 | **Touch is always the fallback.** No workflow requires the buttons | P0 | ✔ | Draft |

Hardware notes (detail in §8):
- **Arcade buttons + USB encoder** (~$15–30): the simplest option. Choose a **keyboard-mode** encoder (e.g. I-PAC-style) or a gamepad-mode encoder, since both are supported (FR-INP-01/02).
- **DIY:** an RP2040 or ESP32 board running keyboard firmware (QMK/KMK) acts as a USB or BLE keyboard. Easy to make wireless.
- **Power:** a USB-C OTG device blocks the tablet's charging port unless you use an **OTG hub with power pass-through** (verify per tablet model). Bluetooth avoids this but needs batteries or power, and adds pairing and drop-out risk.
- **Wet environment:** sealed arcade buttons or IP65 keypads; mount the box away from the steam wand.

#### 4.9.1 Wireless vs wired buttons: trade-off (Expo)
Robustness is the top priority, but the button box needs to sit at the bar, away from the Expo tablet.

| Option | How | Reliability risks | Mitigations | Location convenience | Verdict |
|---|---|---|---|---|---|
| **A. Wired USB** | Arcade buttons + USB encoder → OTG hub with power pass-through → tablet (active USB extension cable if far) | Lowest. Risks: cable damage, the hub/charging combo on some tablets, and a cable run across the bar | Cable channel / under-counter routing; test the specific hub with the specific tablet | ✘ cable tied to the tablet | Most reliable; use if S6 shows wireless isn't good enough |
| **B. Bluetooth HID keypad** (off the shelf, or DIY BLE) | Paired to the Expo tablet as a keyboard | **Highest.** First press after sleep is often lost or slow (BLE power saving); random drops/re-pairing; Android reconnect behaviour varies by tablet; batteries; 2.4 GHz congestion; **no acknowledgement** (a keyboard can't learn whether its press was applied); tied to one tablet, so it can't follow Expo failover | Disable sleep on DIY firmware; mains power; heartbeat via a "keep-alive" key; but no true ack | ✔ | Prototype only, not for production |
| **C. 2.4 GHz dongle keypad** (Logitech-style receiver) | USB receiver in the tablet | Better than BT at reconnecting, but it still occupies the USB port (OTG + power issue again), sleeps, and has no ack | — | ✔ (but dongle on tablet) | Not recommended |
| **D. Wi-Fi button box (ESP32), our own protocol** — **recommended** | Mains-powered ESP32 + arcade buttons + status LED/beeper. Talks to the KDS over the shop LAN with our authenticated protocol (presses with sequence numbers, ack, heartbeat) | Depends on Wi-Fi, but **so does the whole KDS already**, so it adds no new dependency. Custom firmware to maintain. Wi-Fi reconnect after a router reboot takes a few seconds | Ack + LED feedback (FR-INP-13); exactly-once (FR-INP-14); heartbeat + alert (FR-INP-15); no sleep (mains); **can talk to any tablet, so it follows Expo failover** (FR-INP-17); optional Ethernet (ESP32 with PoE/Ethernet) | ✔ anywhere with power | **Best reliability among wireless options**, and the only one with a real "did it work?" signal |
| **E. Hybrid** | Box D firmware also exposes **USB HID** when a cable is plugged in | Fallback if Wi-Fi is down | Same box, plug in a cable in an emergency | ✔ / ✘ | Nice-to-have on top of D |

**Recommendation:**
1. **Prototype quickly with B** (an off-the-shelf BT keypad, ~$20) to settle the button layout and workflow with staff. Treat it as disposable.
2. **Production: D**, an ESP32 Wi-Fi button box, mains powered, with acknowledged presses and status LEDs, plus E (USB fallback) if cheap.
3. **Spike S6** measures A, B and D side by side at the real bar location. If D misses the FR-INP-16 targets, fall back to A.

Why D beats Bluetooth for robustness: wireless itself isn't the main risk. The risk is **not knowing whether a press arrived**. A keyboard protocol (B, C) is fire-and-forget. Our own protocol (D) gives an acknowledgement, dedupe, link monitoring and failover, which makes failures *visible* instead of silent.

⚠ **UX check:** if the box is out of sight of the Expo screen, positional actions ("bump ticket 3") are risky.
Prefer "bump oldest / bump selected" with the box LED and a large on-screen cursor, plus a short undo (FR-INP-07). See Q-INP-2.

---

## 5. Non-functional requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-REL-01 | **No single point of failure on site**: losing any one tablet does not stop the others from syncing | Required |
| NFR-REL-02 | **Internet outage**: tablets keep syncing with each other, and all existing tickets stay fully workable | Required |
| NFR-REL-03 | **No data loss**: each event is durably written (SQLite WAL, fsync) before the UI acknowledges it | Required |
| NFR-REL-04 | App crash or reboot: back to the same screen and state in under 10 s | Required |
| NFR-REL-05 | Cloud outage: same behaviour as an internet outage; the shop never depends on our cloud being up | Required |
| NFR-REL-06 | Clock skew between tablets must not corrupt ordering or stats (use a hybrid logical clock) | Required |
| NFR-PERF-01 | UI interaction latency under 100 ms; 60 fps scrolling with 50+ open tickets | Target |
| NFR-PERF-02 | Order showing on screen within 3 s p95 while online (FR-ORD-02); sync within 500 ms p95 on LAN (FR-SYN-01) | Target |
| NFR-SEC-01 | All LAN traffic authenticated (per-device keys issued at pairing) and encrypted (TLS or Noise) | Required |
| NFR-SEC-02 | Square tokens stored only in the cloud (and optionally the hub), never on ordinary display tablets | ❓ |
| NFR-SEC-03 | No customer PII kept longer than needed (keep names and phone numbers 24 h on device, configurable in the cloud) | Target |
| NFR-OPS-01 | Remote logs and crash reports; device health on the dashboard (battery, last seen, version) | Target |
| NFR-OPS-02 | Staged app updates (one tablet first), never during open hours unless forced | Target |
| NFR-COST-01 | Cloud running cost under $10/month for one shop | Target |

---

## 6. Robustness & failure modes (the core of the design)

### 6.1 Where orders come from: two independent paths
Square POS on the Terminal is a closed app. Orders leave it in only two ways:

| Path | Transport | Works without internet? | Data | Recoverable? |
|---|---|---|---|---|
| **P1: Printer path** (primary) | Terminal → LAN → ingest tablet(s) emulating a network kitchen printer | **Likely**, per Fresh; ⚠ conflicting reports, so verify in M0 | Ticket as text or **bitmap** (needs OCR) | No, so we add redundant ingest tablets plus P2 reconciliation |
| **P2: API path** (backup and source of truth) | Square cloud → webhooks to Moon Cloud, **and** the on-site Orders API poll | No | Fully structured | Yes, by polling since the last cursor |

How they work together:
- **Normal operation:** P1 puts the ticket on screen in about 1 s. P2 arrives a few seconds later, matches the
  ticket (FR-ORD-11), and quietly fixes any parse errors and adds structured metadata such as the customer name, source and catalog IDs.
- **Internet down:** P1 keeps feeding the KDS (if M0 confirms the Terminal still prints offline). When the internet
  returns, P2 back-fills and reconciles.
- **Printer path broken** (ingest tablet asleep or crashed, Wi-Fi glitch): the second ingest tablet covers it, and P2 back-fills anything missed within seconds while online.
- **Both down** (internet out **and** the Terminal won't print offline): fall back per Q-OFF-1.

❓ **Q-OFF-1: Fallback if both paths are down.** Options:
- **(a) Paper fallback:** staff work from Terminal receipts. *Simplest.*
- **(b) Manual quick-entry on a tablet** from the synced Square catalog, matched to Square orders later.
- **(c) Our own POS front-end** (Mobile Payments SDK with a Square Reader). Big scope.
- Recommendation: **(a) for v1**, and revisit after M0 shows how often this case actually happens.

❓ **Q-OFF-2:** Are you open to an extra cellular backup for the router (an LTE failover modem, about $10–20/mo)?
It covers far more outages than any software fallback and helps Square's own payments too.

### 6.2 Failure-mode table
| # | Failure | Effect | Mitigation | Req |
|---|---|---|---|---|
| F1 | Internet down | API path and cloud unreachable | Printer path keeps ingesting over the LAN; LAN sync continues; stats buffered; banner "Internet offline — printer path only"; fallback per Q-OFF-1 | NFR-REL-02 |
| F1b | Ingest tablet asleep, crashed or off Wi-Fi | Printer path partly or fully lost | Two or more ingest tablets (FR-ORD-10); foreground service (FR-ORD-13); API back-fill; **alert on every screen if no ingest printer is healthy** | FR-ORD-10 |
| F1c | OCR/parse error | Wrong or missing item on screen | Confidence marker plus raw image (FR-ORD-12); API reconciliation overwrites it within seconds | FR-ORD-11/12 |
| F1d | Square update changes ticket layout | Parser breaks | Parser is template-driven and versioned; golden-ticket regression tests; API path keeps orders correct | FR-ORD-11 |
| F1e | Ingest tablet's IP changes | Terminal can't reach the printer | DHCP reservation + Device MAC (setup checklist); health check shows "last ticket received" | FR-ADM-06 |
| F2 | Our cloud down (internet OK) | If the cloud is the only ingest path, no new orders | **Two ingest paths**: cloud webhook receiver **and** on-site hub polling the Square API directly (see §7.3) | NFR-REL-05 |
| F3 | Wi-Fi / LAN down, internet OK via cellular | Tablets can't see each other | Cloud relay path (FR-SYN-03) | FR-SYN-03 |
| F4 | Wi-Fi fully down | Each tablet isolated | Each keeps working on its own local state; merges on reconnect (CRDT/event log) | FR-SYN-04/05 |
| F5 | One tablet dies | Its station is gone | Any tablet takes over the role (FR-RTE-04); state lives on every device | NFR-REL-01 |
| F6 | Hub tablet dies (if we use a hub) | No coordinator | Automatic hub election or peer-to-peer design (§7.2) | NFR-REL-01 |
| F7 | App crash / OS reboot | Brief blackout on one screen | Boot-start, state from SQLite, catch up from peers | NFR-REL-04 |
| F8 | Duplicate / out-of-order webhooks | Duplicate tickets | Idempotent on Square order ID + version | FR-ORD-03 |
| F9 | Order edited or refunded after the ticket is shown | Staff make the wrong thing | Visible "CHANGED" or "CANCELLED" state with a chime | FR-ORD-05 |
| F10 | Tablet clock wrong | Bad ordering or stats | Hybrid logical clocks; cloud and Square timestamps used for stats | NFR-REL-06 |
| F11 | Square OAuth token expires or is revoked | Ingest silently stops | Auto-refresh; loud alert on every screen and email to owner | FR-ADM-05 |
| F12 | Router reboots and IPs change | Peers lost | mDNS rediscovery; peers identified by device key, not IP | FR-SYN-06 |

---

## 7. Architecture (proposed)

### 7.1 Overview
```
                     ┌──────────────────── Square Cloud ────────────────────┐
                     │  Orders API · Webhooks · Catalog API · OAuth         │
                     └──────┬───────────────────────────────▲───────┬───────┘
                   webhooks │                     sync up   │       │ P2: API poll
                            ▼                               │       │ (backup path)
              ┌──────────────────────────┐                  │       │
              │   Moon Cloud             │                  │       │
              │ • webhook receiver (P2)  │                  │       │
              │ • event store / stats DB │                  │       │
              │ • config & backup        │                  │       │
              │ • relay (WebSocket)      │                  │       │
              │ • web dashboard          │                  │       │
              └──────────┬───────────────┘                  │       │
             WSS (relay, │ stats upload, config)            │       │
   ══════════════════════╪══════════ shop internet ═════════╪═══════╪══════
                         │                                  │       │
  ┌──────────────────────┴──────── shop LAN (Wi-Fi) ────────┴───────┴──────┐
  │                                                                        │
  │   ┌─────────────────┐   P1: "print" kitchen ticket (TCP 9100)          │
  │   │ Square Terminal │────────────┬───────────────────┐                 │
  │   └─────────────────┘            ▼                   ▼                 │
  │                         ┌────────────────┐   ┌────────────────┐        │
  │                         │ Ingest tablet 1│   │ Ingest tablet 2│        │
  │                         │ (e.g. Expo)    │   │ (e.g. Hot bar) │        │
  │                         │ printer emu +  │   │ printer emu +  │        │
  │                         │ parser/OCR     │   │ parser/OCR     │        │
  │                         └───────┬────────┘   └───────┬────────┘        │
  │                                 │  OrderReceived     │                 │
  │                                 ▼  events (deduped)  ▼                 │
  │   ◀════════ peer-to-peer event-log sync (WebSocket, mDNS) ════════▶    │
  │   ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐    │
  │   │ Tab 3  │ │ Tab 4  │ │ Tab 5  │ │ Tab 6  │ │ Tab 7  │ │ Tab 8  │    │
  │   └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ └────────┘    │
  │   every tablet: full replica (SQLite) + routing rules → its stations   │
  └────────────────────────────────────────────────────────────────────────┘
```
Square only knows about **one or two printers**. Moon KDS decides which tablet shows which item.

### 7.2 Sync model: replicated event log (recommended)
- All state changes are **immutable events**: `OrderReceived`, `OrderUpdated`, `ItemBumped`,
  `StationBumped`, `OrderRecalled`, `ConfigChanged`, and so on. Each event carries
  `{event_id (UUIDv7), device_id, hlc_timestamp, payload}`.
- Each tablet stores the **full event log** for the business day plus a materialised view (current tickets) in SQLite.
- Sync is **anti-entropy**: peers exchange version vectors (`device_id → highest seq seen`) and send
  each other the missing events. Gossip runs over WebSocket on the LAN, and through the cloud relay when it's reachable.
- Ticket state is derived **deterministically** from the event set (a CRDT-style merge). For example, bump
  versus recall is resolved by HLC order, so every device converges no matter what order events arrive in.
- **Why not a leader/hub?** A single hub is simpler but is a single point of failure (F6), and hub
  election on flaky Wi-Fi is risky. With peer-to-peer replication, any surviving tablet carries the
  whole state. The event volume is tiny (a few thousand events a day), so a full replica on every device costs almost nothing.

❓ **Q-ARCH-1:** Agree with peer-to-peer replication over a single hub tablet? (Recommended: P2P.)

### 7.3 Order ingestion

#### 7.3.1 Printer path (P1): Square-routed vs Moon-routed
| | **Square-routed** (Fresh style) | **Moon-routed** (recommended) |
|---|---|---|
| Square printer stations | One per tablet (8 today) | **1–2 ingest stations, "all categories"** |
| Where routing lives | Square categories per printer | Moon KDS rules, synced over the LAN |
| Add or move a tablet | Edit Square on the Terminal | Pick a station on the tablet; Square untouched |
| Item- or modifier-level routing | Category only | Category, item, modifier, order type |
| Expo sees the whole order | Needs its own "all" printer | Built in |
| Terminal print load per order | Up to 8 jobs over Wi-Fi | 1–2 jobs |
| Depends on inter-tablet sync | No | **Yes** (it's P2P, local-first, and has no single point of failure, see §7.2) |
| If an ingest tablet dies | Only its station is lost | Covered by the second ingest tablet |

Both can run at once (FR-RTE-08). Moon-routed is the default.

#### 7.3.2 Printer emulation (needs the M0 spike)
- **Discovery:** answer Square's printer discovery for one supported model (Star LAN, e.g. TSP143IIILAN or TSP100IV,
  on UDP 22222, or Epson TM-m30-class), or rely on Square's "Advanced Printer Setup" by IP and skip discovery.
- **Job intake:** TCP 9100 (raw/StarPRNT/ESC-POS), plus whatever status channel the model needs (Star uses TCP 9101 status).
  The ack must look like a real printer's, or Square will show "printer error".
- **Decode:** text ESC/POS → parse directly; raster → render to bitmap → **on-device OCR**
  (ML Kit Text Recognition, offline) → template parser (header, ticket #, name, order type, items, indented modifiers, notes).
- **Dedupe key:** Square ticket/order number + time window + content hash (FR-ORD-10).
- **Reconcile with P2:** match on the ticket number shown on the Square order (`ticket_name`/display number) plus time.
  Each field is tagged by source; API data wins.
- **Legal / ToS:** emulating a generic printer protocol for interoperability is our working assumption. We **do not**
  use or imitate the "Fresh KDS" model. ❓ **Q-LEGAL-1:** check the Square seller ToS before the pilot.

#### 7.3.3 API path (P2)
1. **Cloud webhook:** Square webhook (`order.created`, `order.updated`, `order.fulfillment.updated`, `payment.*`)
   → Moon Cloud → fetch full order → `OrderReceived` event → pushed to tablets over the relay.
2. **On-site poll:** one or more tablets marked "ingestor" poll the Square
   Orders API (`SearchOrders`, updated since the last cursor) every 2–5 s while online, and write the same `OrderReceived` events.
   This keeps orders flowing **when our cloud is down but the internet is up** (F2).
- Both paths produce events keyed on `square_order_id + version`, so duplicates collapse (FR-ORD-03).
- ❓ **Q-ARCH-2:** The on-site path means a Square access token lives on at least one tablet. It's acceptable with
  Android Keystore encryption, but it's a trade-off (NFR-SEC-02). Option: a small dedicated on-site box
  (Raspberry Pi or mini-PC) as the ingestor and an always-on peer instead of a tablet.
- Check during the Square API spike: polling rate limits and the real webhook latency.

### 7.4 Cloud component
- **Responsibilities:** webhook receiver, Square OAuth, long-term event and stats store, config master copy,
  WebSocket relay, web dashboard, device registry and health, optional SMS.
- **The cloud is never in the critical path for the shop LAN.** Tablets treat it as one more peer.
- Candidate stacks (❓ **Q-CLOUD-1**):
  - **Cloudflare Workers + Durable Objects + D1:** WebSocket relay is natural with Durable Objects, very cheap, always on.
  - **Supabase (Postgres + Realtime + Edge Functions):** good SQL for stats, built-in auth.
  - **Small VPS (Fly.io or Hetzner) running a Kotlin/Ktor server + Postgres:** shares code with the Android app.
  - Recommendation: **Ktor on Fly.io or a small VPS**, which shares the Kotlin core with the tablets (see §7.5.4). Cloudflare stays an option for the relay. Decide after the spike.

### 7.5 Android app: technology choice

#### 7.5.1 What the stack must do well (from the requirements)
1. **Raw networking:** a TCP 9100 server and UDP discovery responder for printer emulation (FR-ORD-09); LAN WebSocket server and client; mDNS.
2. **Always on in the background:** a foreground service that keeps the printer listener and sync engine alive when the screen sleeps (FR-ORD-13), plus boot start and kiosk / Device Owner (FR-ADM-04).
3. **Hardware input:** keyboard **and gamepad** key events, long-press, and focus handling (FR-INP-*).
4. **On-device OCR** if tickets turn out to be raster (ML Kit, offline).
5. **Rendering:** smooth with 50+ tickets and many styled pills (FMT) on a cheap tablet.
6. **A deterministic, heavily tested core:** event log, HLC, reducers, sync protocol, ticket parser and formatting rules engine. Ideally shared with the cloud.
7. Maintainable by a small team for years, with good tooling and AI-assist support.

#### 7.5.2 Options compared
Score: ●●● strong · ●● adequate (some extra work) · ● weak (significant custom native work) · ✘ not possible

| Criterion | **Kotlin native** (Jetpack Compose) | **Kotlin Multiplatform** (shared core + Compose) | **React Native** (TypeScript, New Architecture / Expo) | **Flutter** (Dart) | **.NET MAUI** (C#) | **Web / PWA** in a kiosk browser (or Capacitor) | **Java native** (Views) |
|---|---|---|---|---|---|---|---|
| TCP 9100 server / UDP discovery | ●●● platform APIs | ●●● | ●● community libs (`react-native-tcp-socket`, `-udp`); custom native module for anything unusual | ●●● `dart:io` sockets built in | ●●● .NET sockets | ✘ browsers cannot listen on sockets (Capacitor: native plugin needed) | ●●● |
| mDNS (NsdManager) | ●●● | ●●● | ●● `react-native-zeroconf` | ●● `nsd` / `multicast_dns` | ●● bindings | ✘ | ●●● |
| Foreground service, survives screen off | ●●● | ●●● | ● JS runtime is unreliable in the background; ends up needing a native Kotlin service anyway | ●● `flutter_foreground_task`, background isolate | ●● | ✘ | ●●● |
| Kiosk / Device Owner / boot start | ●●● | ●●● | ●● native module | ●● plugin / native code | ●● | ● managed browser kiosk only | ●●● |
| Keyboard **+ gamepad** input | ●●● full `KeyEvent` / `InputDevice` | ●●● | ● keyboard via lib; **gamepad needs a custom native module** | ●● keyboard good; gamepad via plugin | ●● | ●● Keyboard + Gamepad API in Chrome | ●●● |
| On-device OCR (ML Kit) | ●●● first-party | ●●● | ●● wrapper libs | ●● `google_mlkit_text_recognition` | ● | ● (Tesseract.js, slow) | ●●● |
| UI performance on cheap tablets | ●●● | ●●● | ●● good with the New Architecture; long lists need care | ●●● | ●● | ●● | ●● verbose |
| Dev speed / UI iteration | ●● | ●● | ●●● fast refresh, huge ecosystem | ●●● hot reload | ●● | ●●● | ● |
| Over-the-air updates without the Play Store | ●● self-hosted APK updates via Device Owner | ●● | ●●● EAS Update / CodePush for JS changes | ●● (Shorebird) | ● | ●●● instant | ●● |
| Share core logic with the cloud | ●● (Ktor server in Kotlin) | ●●● **same code** on device and server | ●●● TypeScript on device, Workers and dashboard | ● Dart servers are niche | ●● ASP.NET | ●●● | ●● |
| Share UI with the web dashboard | ✘ | ● (Compose for Web, immature) | ●● React concepts, separate components | ●● Flutter Web (meh for dashboards) | ● | ●●● same app | ✘ |
| Testability of the sync core (deterministic simulation, property tests) | ●●● JVM tests, Kotest | ●●● | ●● Jest + fast-check | ●● | ●● | ●● | ●● |
| Native code needed for *our* requirements | none | none | **many modules** (service, sockets, gamepad, kiosk, maybe OCR) | some plugins | some | **most features impossible** | none |
| Long-term risk | low (Google first-party) | low–med (KMP stable; Compose MP newer) | med (dependency churn, RN upgrades) | low–med | med–high (MAUI maturity) | low for UI, but disqualified for ingest | low, but legacy style |
| iOS later (non-goal) | ✘ | ●● | ●●● | ●●● | ●● | ●●● | ✘ |

#### 7.5.3 Takeaways
- **Web/PWA is out** for ingest and sync tablets (no listening sockets, no background service). It could still serve a read-only customer status board (FR-CUS-01) or the dashboard.
- **React Native** is excellent for typical app UIs, but this project's hardest parts (socket server, always-on service, gamepad input, kiosk, possibly OCR) all sit **outside** what RN does natively. We would write and maintain those in Kotlin anyway, then bridge them into JS, with **two languages and a bridge in the critical path**. Its strongest advantages (OTA updates, TypeScript shared with a web dashboard) are real but don't outweigh this.
- **Flutter** is the strongest cross-platform option here (sockets built in, good performance, good keyboard handling), but it still needs plugins for the service, kiosk and gamepad, and Dart is weak for sharing logic with the cloud.
- **Kotlin native** has zero gaps against the requirements. Every hard requirement is a first-party Android API.

#### 7.5.4 Recommendation
**Kotlin, with Jetpack Compose for the UI, and the core as a Kotlin Multiplatform (pure Kotlin) module.**
- `core-*` modules (events, HLC, reducers, sync protocol, ticket parser, formatting rules engine) are pure Kotlin with no Android
  dependencies. They're unit- and simulation-tested on the JVM, and **reused unchanged in the cloud server (Ktor)**, so reconciliation and stats use the
  exact same logic as the tablets.
- The Android app is a thin shell: Compose UI, foreground service, sockets, NsdManager, input, ML Kit.
- **Web dashboard:** a separate TypeScript/React app talking to the cloud's JSON API. The UI isn't shared, which is fine because a dashboard and a KDS screen are different products.
- This tilts **Q-CLOUD-1** toward **Ktor on a small VPS/Fly.io**, so the cloud server can share the Kotlin core. Cloudflare stays an option for the relay only.
- Runner-up: **Flutter**, if fast UI iteration or iOS ever becomes more important than first-party Android access.

Implementation notes:
- Min SDK 26 (Android 8), target the latest version.
- **SQLite** through SQLDelight (multiplatform), so the cloud and tests can share schema and queries.
- **Ktor** embedded server and client for LAN WebSockets, and **NsdManager** (mDNS) for discovery.
- Foreground service keeps the printer listener and sync engine alive. `BOOT_COMPLETED` starts the app, and lock-task (kiosk) mode is set up through Android Device Owner provisioning.
- Module layout (proposed):
  `core-model` (events, reducers, HLC) · `core-sync` · `core-format` (rules engine) · `core-ticket` (printer-stream parser) ·
  `android-ingest` (printer emulation, OCR, Square poll) · `android-input` (key/gamepad mapping) · `data` (SQLDelight) · `ui-kds` · `ui-admin` · `app` · `cloud-server` · `firmware-buttonbox` (ESP32, ESP-IDF/Arduino C++; paired like a tablet at setup, same authenticated LAN protocol per NFR-SEC-01).

✔ **Q-TECH-1 resolved (D-008):** Kotlin native + a shared Kotlin core. The owner has no language preference.

### 7.6 Data model (sketch)
```
Order        { id, square_order_id, square_version, display_number, customer_name,
               source, fulfillment_type, pickup_at, created_at, notes, state }
LineItem     { id, order_id, catalog_item_id, name, qty, modifiers[], note, station_ids[], state }
Station      { id, name, routing_rules[], timer_thresholds, sound, layout }
FormatRule   { id, priority, enabled, scope (all | station_ids[] | device_ids[]),
               match { kind: modifier|modifier_group|item|category|note_keyword|order_type,
                       catalog_id?, name_normalised?, regex? },
               style { text_color?, bg_color?, bold?, uppercase?, size?, badge?, tint: item|ticket?, blink_s? },
               display { rename?, hide?, sort_rank? }, sound? }
KeyMapping   { device_id, input_device_descriptor, key_or_button_code, press: short|long|double, action, arg? }
Device       { id, name, public_key, assigned_station_ids[], role_flags (ingestor, expo…) }
Event        { event_id, device_id, hlc, type, payload }   // the only thing that is synced
```
Order states: `NEW → IN_PROGRESS → READY (per station) → COMPLETED (expo)`, plus `CANCELLED`, and `RECALLED` goes back to the previous state.

---

## 8. Hardware

| Item | Proposal | ❓ |
|---|---|---|
| Tablets | Samsung Galaxy Tab A9+ / Lenovo Tab M11 class, Android 13+, 11" | Already own any? |
| Mounts | VESA or under-cabinet arms, splash-resistant cases | |
| Power | Always-on chargers; battery-protection mode (charge to 85%) where the device supports it | |
| Network | Dedicated SSID or VLAN for KDS with client-to-client traffic allowed; ideally a UPS on the router | |
| Optional | Raspberry Pi 5 / mini-PC as an always-on peer and ingestor (Q-ARCH-2) | |
| Buttons | **Expo only (v1).** Production: a mains-powered ESP32 Wi-Fi button box (3–5 sealed 60 mm arcade buttons, RGB status LED, beeper, optional USB-HID fallback). Prototype: an off-the-shelf BT keypad. Wired USB + powered OTG hub is the fallback if S6 fails (§4.9.1) | Mains power at the bar spot? |

---

## 9. Milestones (draft)

| M | Scope | Exit criteria |
|---|---|---|
| M0 | **Design sign-off** (this doc) + **Square spikes** (below) | All P0 ❓ resolved; go/no-go on the printer path |

**M0 spikes** (need the real Terminal, one spare Android tablet, and a laptop on the shop LAN):
1. **S1, what does Square send?** Add emulated LAN printers and capture the bytes for a test print and real orders. Which models does the Terminal offer? Text or raster? Try in this order:
   - (a) **Impact (dot-matrix) kitchen printers, e.g. Epson TM-U220 class.** These are poor at graphics, so Square very likely sends them **plain text**. That would avoid OCR entirely. ⚠ First confirm the model is on Square's supported list for the Terminal.
   - (b) Epson thermal (TM-m30/TM-T88 class).
   - (c) Star thermal (TSP100/TSP143 LAN).
   - (d) **The "Fresh KDS" printer entry:** what does Square send to it, and how does it find the device (mDNS/Bonjour, UDP broadcast, IP)? Is there any handshake or status exchange?
   - **Every printer model the Terminal offers** gets a captured, documented sample ticket in `docs/protocols/` for posterity, whatever we end up using.
2. **S2, offline printing:** unplug the router's WAN, take orders on the Terminal in Offline Mode. Does it still print to LAN printers?
3. **S3, OCR accuracy** (if raster): 200+ real tickets incl. long modifier lists; target ≥ 99.5% exact line accuracy after the template parser.
4. **S4, API timing:** webhook and poll latency vs the printer path; confirm a field that reliably links a printed ticket to its Square order.
5. **S5, multi-printer behaviour:** Terminal with 2 ingest printers. Are both printed in parallel? Is there a retry if one is down? Error pop-ups on the Terminal?
6. **S6, button reliability:** at the real bar location during service hours, compare wired USB, a BT keypad and an ESP32 Wi-Fi prototype. Run 1,000 presses each, including presses after 5/30/120 min idle, a microwave or blender running, and a router reboot. Measure missed and duplicate presses, latency, and reconnect time against FR-INP-16.

| M | Scope | Exit criteria |
|---|---|---|
| M1 | `core-model` + `core-sync`: event log, HLC, reducers, LAN sync between emulators | Property tests: random event orders converge |
| M2 | Single-station KDS UI fed by the **printer path** (ingest + parse/OCR), plus the API poll for reconciliation | Orders on screen < 2 s; bump/recall; reconciliation fixes injected parse errors |
| M3 | Multi-station routing, expo, multi-tablet LAN sync, failover, **formatting rules editor (FMT P0)**, **key/gamepad mapping + arcade buttons (INP P0)** | Chaos test: kill/rejoin tablets, Wi-Fi off/on; a full rush shift run with buttons only |
| M4 | Cloud: webhook ingest, relay, stats upload, minimal dashboard | Offline buffer replays with no loss |
| M5 | Pilot in the shop alongside paper or Fresh; fix what the bar finds | 2 weeks with no lost tickets |
| M6 | Parity extras: all-day counts, customer board, SMS, bump bar, reports | Fresh parity matrix fully green for P1 |

---

## 10. Testing strategy
- **Deterministic simulation tests** for sync: N virtual devices, random network partitions, delays,
  duplicate deliveries and reboots; check that every device converges and no event is lost.
- **Property-based tests** on reducers (event order independence where required).
- **Square sandbox** integration tests; recorded webhook fixtures.
- **Chaos drills on real hardware** before the pilot: pull router power mid-rush simulation, kill the app, airplane-mode a tablet.
- **Performance test:** synthetic 150 orders/hour for 3 h on the cheapest target tablet.

---

## 11. Open questions (consolidated)

| ID | Question | Blocks |
|---|---|---|
| Q-BENCH-1 | Fresh KDS experience: what you like and dislike | §4 priorities |
| Q-A-01..06 | Confirm shop assumptions (§3) | Sizing, routing |
| Q-OFF-1 | Fallback when both the printer and API paths are down: paper / manual entry / own POS | Scope of M2–M6 |
| Q-ING-1 | Moon-routed (1–2 ingest printers in Square) vs Square-routed (one printer per tablet)? Recommended: Moon-routed | M2 |
| Q-ING-2 | Which tablets are ingest tablets? (e.g. Expo + one bar tablet, on different power circuits) | M3 |
| Q-ING-3 | Is Fresh KDS live in the shop now? Can we run the spike alongside it (separate printer station) without disrupting service? | M0 |
| Q-LEGAL-1 | Square ToS check on printer emulation, **especially if production relies on the "Fresh KDS" entry** (it carries a third party's name in the Square UI). Decide after S1 shows which formats are viable | Pilot |
| Q-PART-1 | Ask Square Developer Relations whether a KDS can join the same partner printer/KDS integration Fresh uses (the legitimate route to a structured LAN feed) | M0, in parallel |
| Q-OFF-2 | Cellular failover for the router? | F1 |
| Q-ARCH-1 | Peer-to-peer replication vs hub | M1 |
| Q-ARCH-2 | Square token on a tablet vs dedicated on-site box | M2, NFR-SEC-02 |
| Q-CLOUD-1 | Cloud stack | M4 |
| ~~Q-TECH-1~~ | ✔ Resolved: Kotlin (D-008) | — |
| ~~Q-FMT-1~~ | ✔ Resolved: milk and espresso types on day one (D-009). Still to do: confirm the exact modifier lists and colours from the Square catalog | M3 |
| ~~Q-INP-1~~ | ✔ Resolved: Expo only; wireless preferred if it passes S6 (D-010) | — |
| Q-INP-2 | Where exactly will the button box sit, and can staff see the Expo screen from there? Which actions does Expo need (e.g. bump oldest, next/prev, recall)? How many buttons? | M3 |
| Q-INP-3 | Is there mains power (an outlet or USB) at the button location? Is Ethernet nearby (for a PoE/Ethernet box)? | M3 |
| Q-CUS | Customer status board / SMS wanted? | M6 |
| Q-HW | Existing tablets, bump bars? | M5 |

---

## 12. Decision log

| Date | ID | Decision | Rationale |
|---|---|---|---|
| 2026-10-30 | D-001 | Fresh KDS is the main benchmark | Owner direction |
| 2026-10-30 | D-002 | Robustness / offline operation is the top priority, above feature breadth | Owner direction |
| 2026-10-30 | D-003 | One Square Terminal; 8 tablets now, count variable | Owner input |
| 2026-10-30 | D-004 | **Primary ingest = tablet acts as a LAN network printer for the Terminal** (like Fresh printer mode, which uses Wi-Fi/LAN, not Bluetooth). Square API is the backup and reconciliation path | Owner direction; only path that can work without internet |
| 2026-10-30 | ~~D-005~~ | ~~Do not use or imitate the "Fresh KDS" printer model~~: **superseded by D-006** | Was based on an assumed handshake or licence check, for which no evidence was found |
| 2026-10-30 | D-006 | Spike S1 captures and documents the message format of **every** printer model the Terminal offers, **including the "Fresh KDS" entry**. The production ingest format is chosen from the results (text over raster; simplest robust option) | Evidence suggests the Fresh entry is a lightly modified printer stream, probably the most decodable; decide on data, not assumption |
| 2026-11-01 | D-007 | Custom formatting rules (FMT) and physical button input (INP) are P0 requirements | Owner direction |
| 2026-11-01 | D-008 | **Kotlin** (Jetpack Compose UI + shared pure-Kotlin core, Ktor cloud) | Owner has no language preference; Kotlin has no gaps against the requirements (§7.5) |
| 2026-11-01 | D-009 | Day-one formatting presets: **milk types and espresso types** | Owner input |
| 2026-11-01 | D-010 | Physical buttons on the **Expo tablet only** to start. Wireless preferred for placement at the bar, **if** it passes the S6 reliability spike; recommended design is an ESP32 Wi-Fi box with acknowledged presses | Owner input; robustness first |

---

## 13. Revision history
| Rev | Date | Notes |
|---|---|---|
| 0.1 | 2026-10-30 | First draft: requirements skeleton, failure modes, proposed architecture |
| 0.2 | 2026-10-30 | Researched how Fresh works with Square (printer mode over LAN vs Orders API). Printer path becomes primary ingest, with redundancy, OCR risk and reconciliation. Moon-routed vs Square-routed comparison. M0 spikes S1–S5. Assumptions A-01/A-02 confirmed |
| 0.3 | 2026-10-30 | Fresh printer-entry evidence reviewed: no handshake or licence evidence found; Fresh began as an Epson printer emulator. D-005 superseded by D-006: S1 captures every printer type, including the Fresh entry |
| 0.4 | 2026-11-01 | Added §4.8 custom formatting rules (FMT), §4.9 peripheral input / arcade buttons (INP), and the technology comparison matrix plus recommendation (§7.5) |
| 0.5 | 2026-11-01 | Resolved Q-TECH-1 (Kotlin), Q-FMT-1 (milk + espresso), Q-INP-1 (Expo only). Added the wireless-vs-wired button trade-off (§4.9.1), FR-INP-12..19 and spike S6 |
