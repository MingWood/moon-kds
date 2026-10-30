# Moon KDS — Design Document

| | |
|---|---|
| **Status** | DRAFT, revision 0.1 (first pass, not yet reviewed) |
| **Owner** | Moonwake Roasters |
| **Last updated** | 2026-10-30 |
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

Fresh KDS is an Android KDS app with Square integration. Its main traits, from public material
(⚠ check details against a live trial before relying on them):

- Orders come from Square through Square's cloud APIs/webhooks, so **Fresh needs the internet to
  receive new Square orders**.
- Screens sync with each other, and the cloud, through Fresh's backend. Local-network operation is
  limited.
- Features: routing by category/item to screens, expo screen, item-level and order-level bumping,
  recall, colour-coded timers, all-day item counts, modifier highlighting, bump bar support, customer
  SMS notifications, customer-facing order status screen, prep-time reports in a web dashboard,
  and per-screen layout options.

**Where we aim to beat it:** a local-first design so that devices already on the shop network keep
syncing during an outage, plus a defined degraded mode for taking orders while Square is unreachable
(see §6).

❓ **Q-BENCH-1:** Do you currently use (or have you trialled) Fresh KDS? Which features do you rely on
and which annoy you? That will turn §4 from "parity" into "parity plus fixes".

---

## 3. Shop context & assumptions

| ID | Assumption | Confirm? |
|---|---|---|
| A-01 | 1 Square Terminal (maybe more at peak) running Square Point of Sale | ❓ |
| A-02 | 2–4 Android tablets, 10" or larger, wall- or counter-mounted, always powered | ❓ |
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
| FR-ORD-07 | Manual "offline order" entry on any tablet when Square is unreachable (see §6.3) | P1 | ✘ | ❓ |
| FR-ORD-08 | Orders taken on the Terminal in Square's offline mode are ingested once Square uploads them, flagged "late" and not double-counted | P0 | ? | Draft |

### 4.2 Routing & stations (RTE)
| ID | Requirement | Pri | Fresh | Status |
|---|---|---|---|---|
| FR-RTE-01 | Route line items to stations by Square category, item, or modifier rules | P0 | ✔ | Draft |
| FR-RTE-02 | One order can split across several stations, each seeing only its own items (with an optional "other items" hint) | P0 | ✔ | Draft |
| FR-RTE-03 | Expo station sees the whole order and the bump status from every prep station | P0 | ✔ | Draft |
| FR-RTE-04 | Any tablet can take over any station's role (failover) in two taps or fewer | P0 | ~ | Draft |
| FR-RTE-05 | A tablet can show more than one station (e.g. a quiet day runs on one screen) | P1 | ✔ | Draft |
| FR-RTE-06 | Routing rules are edited on a tablet or the cloud dashboard and synced everywhere | P1 | ✔ | Draft |

### 4.3 Display & interaction (UI)
| ID | Requirement | Pri | Fresh | Status |
|---|---|---|---|---|
| FR-UI-01 | Ticket grid (tiles) layout, plus an optional rail/list layout | P0 | ✔ | Draft |
| FR-UI-02 | Elapsed-time timer per ticket with colour thresholds (e.g. green, then yellow at 3 min, red at 5 min), configurable per station | P0 | ✔ | Draft |
| FR-UI-03 | Bump a whole order with one tap; bump single items by tapping an item | P0 | ✔ | Draft |
| FR-UI-04 | Recall / undo the last N bumped orders (at least the last 20, or last 30 min) | P0 | ✔ | Draft |
| FR-UI-05 | Highlight modifiers, allergies and notes (e.g. "OAT", "EXTRA HOT", allergy keywords in red) | P0 | ✔ | Draft |
| FR-UI-06 | Show order number or customer name, source (in-store, online, delivery) and dine-in or to-go | P0 | ✔ | Draft |
| FR-UI-07 | All-day counts: running totals of items outstanding (e.g. "6× Oat Latte") | P1 | ✔ | Draft |
| FR-UI-08 | Audible chime on a new order, configurable per station | P0 | ✔ | Draft |
| FR-UI-09 | Item consolidation: identical items with identical modifiers grouped as "2×" | P1 | ✔ | Draft |
| FR-UI-10 | Mark in progress / started (e.g. tap the header), visible to other stations | P1 | ~ | Draft |
| FR-UI-11 | Prioritise or rush a ticket; pin it to the front | P2 | ? | Draft |
| FR-UI-12 | Hardware bump bar or USB/Bluetooth keypad support | P2 | ✔ | ❓ |
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

### 6.1 The hard constraint: where orders come from
Square Point of Sale on the Terminal is a closed app. Third-party apps **cannot** read orders from it
directly over the LAN. Orders reach outside systems **only through Square's cloud** (webhooks and
the Orders API). So:

> **If the shop's internet is down, no Square order can reach any KDS, ours or Fresh's.**
> Square's own offline mode stores payments on the Terminal and uploads them later.

What we *can* guarantee during an outage:
1. Every ticket already on the screens stays fully workable, and bumps keep syncing over the LAN.
2. Orders keep flowing again as soon as the internet returns, including ones Square queued offline (FR-ORD-08).
3. A defined way to keep making drinks during the outage (pick one in Q-OFF-1 below).

❓ **Q-OFF-1: How should orders get to the bar during an internet outage?** Options:
- **(a) Paper fallback:** Square's offline mode prints the receipt or ticket on the Terminal, and staff work from paper. We add nothing. *Simplest.*
- **(b) Manual quick-entry on a tablet:** a fast menu (synced from the Square catalog) on the expo or register tablet. Staff ring it on the Terminal for payment and tap it into the KDS. When Square catches up, we match the Square order to the manual ticket. *Medium effort; matching can be fuzzy.*
- **(c) Our own POS front-end:** a custom Android POS app on a tablet that takes the payment through Square (Mobile Payments SDK with a Square Reader, which supports offline payments) and writes orders straight into the local KDS. *Most robust, but replaces the Terminal workflow. Big scope.*
- Recommendation: **(a) for v1, design for (b) in v1.x**. Keep (c) as a long-term option.

❓ **Q-OFF-2:** Are you open to an extra cellular backup for the router (an LTE failover modem, about $10–20/mo)?
It covers far more outages than any software fallback and helps Square's own payments too.

### 6.2 Failure-mode table
| # | Failure | Effect | Mitigation | Req |
|---|---|---|---|---|
| F1 | Internet down | No new Square orders; cloud unreachable | LAN sync continues; stats buffered; banner "Square offline — orders will resume"; fallback per Q-OFF-1 | NFR-REL-02 |
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
  Square Terminal ──▶│  Orders API · Webhooks · Catalog API · OAuth         │
                     └──────┬───────────────────────────────┬───────────────┘
                   webhooks │                               │ poll (backup path)
                            ▼                               │
              ┌──────────────────────────┐                  │
              │   Moon Cloud             │                  │
              │ • webhook receiver       │                  │
              │ • event store / stats DB │                  │
              │ • config & backup        │                  │
              │ • relay (WebSocket)      │                  │
              │ • web dashboard          │                  │
              └──────────┬───────────────┘                  │
             WSS (relay, │ stats upload, config)            │
   ══════════════════════╪══════════ shop internet ═════════╪══════════
                         │                                  │
        ┌────────────────┴──────── shop LAN (Wi-Fi) ────────┴───────┐
        │                                                           │
        │  ┌───────────┐   ┌───────────┐   ┌───────────┐  ┌───────┐ │
        │  │ Tablet A  │◀─▶│ Tablet B  │◀─▶│ Tablet C  │◀▶│ Expo  │ │
        │  │ Hot bar   │   │ Cold bar  │   │ Food      │  │       │ │
        │  └───────────┘   └───────────┘   └───────────┘  └───────┘ │
        │     every tablet: full replica (SQLite) + sync engine     │
        └───────────────────────────────────────────────────────────┘
```

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

### 7.3 Order ingestion: two independent paths
1. **Cloud path (primary):** Square webhook (`order.created`, `order.updated`, `order.fulfillment.updated`, `payment.*`)
   → Moon Cloud → fetch full order → `OrderReceived` event → pushed to tablets over the relay.
2. **On-site path (backup and low-latency):** one or more tablets marked "ingestor" poll the Square
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
  - Recommendation: Cloudflare (cost and uptime) or Ktor on Fly (code sharing). Decide after the spike.

### 7.5 Android app
- **Kotlin + Jetpack Compose**, min SDK 26 (Android 8), target the latest version.
- **SQLite** through Room or SQLDelight (SQLDelight if the sync core moves into Kotlin Multiplatform shared with the cloud).
- **Ktor** embedded server and client for LAN WebSockets, **NsdManager** (mDNS) for discovery.
- Foreground service keeps the sync engine alive; `BOOT_COMPLETED` starts the app; lock task (kiosk) mode
  through Android Device Owner provisioning.
- Module layout (proposed):
  `core-model` (events, reducers, HLC; pure Kotlin, heavily unit-tested) · `core-sync` · `square-ingest` ·
  `data` (SQLite) · `ui-kds` · `ui-admin` · `app`.

❓ **Q-TECH-1:** Any preference on language or stack? (Kotlin native is recommended for kiosk control,
background services and performance. Flutter or React Native would be the alternatives.)

### 7.6 Data model (sketch)
```
Order        { id, square_order_id, square_version, display_number, customer_name,
               source, fulfillment_type, pickup_at, created_at, notes, state }
LineItem     { id, order_id, catalog_item_id, name, qty, modifiers[], note, station_ids[], state }
Station      { id, name, routing_rules[], timer_thresholds, sound, layout }
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
| Optional | Raspberry Pi 5 / mini-PC as an always-on peer and ingestor (Q-ARCH-2); USB bump bar (FR-UI-12) | |

---

## 9. Milestones (draft)

| M | Scope | Exit criteria |
|---|---|---|
| M0 | **Design sign-off** (this doc) + **Square API spike** (webhook latency, polling, offline-order behaviour) | All P0 ❓ resolved |
| M1 | `core-model` + `core-sync`: event log, HLC, reducers, LAN sync between emulators | Property tests: random event orders converge |
| M2 | Single-station KDS UI fed by Square sandbox orders (on-site polling) | Orders on screen < 3 s; bump/recall |
| M3 | Multi-station routing, expo, multi-tablet LAN sync, failover | Chaos test: kill/rejoin tablets, Wi-Fi off/on |
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
| Q-OFF-1 | Order fallback during an internet outage: paper / manual entry / own POS | Scope of M2–M6 |
| Q-OFF-2 | Cellular failover for the router? | F1 |
| Q-ARCH-1 | Peer-to-peer replication vs hub | M1 |
| Q-ARCH-2 | Square token on a tablet vs dedicated on-site box | M2, NFR-SEC-02 |
| Q-CLOUD-1 | Cloud stack | M4 |
| Q-TECH-1 | App stack (Kotlin native recommended) | M1 |
| Q-CUS | Customer status board / SMS wanted? | M6 |
| Q-HW | Existing tablets, bump bars? | M5 |

---

## 12. Decision log

| Date | ID | Decision | Rationale |
|---|---|---|---|
| 2026-10-30 | D-001 | Fresh KDS is the main benchmark | Owner direction |
| 2026-10-30 | D-002 | Robustness / offline operation is the top priority, above feature breadth | Owner direction |

---

## 13. Revision history
| Rev | Date | Notes |
|---|---|---|
| 0.1 | 2026-10-30 | First draft: requirements skeleton, failure modes, proposed architecture |
