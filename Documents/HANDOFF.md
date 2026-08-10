# Handoff — Harvesta openHAB, session of 2026-08-10

Written for a fresh agent session with no memory of this work. Read `CLAUDE.md` and
`PROJECT_MEMORY.md` first for the standing project rules; this file covers **what changed
today and what is still open**. Data coverage per season is in `data_inventory.md`.

---

## 1. The single most important thing

**The OH5 dev instance is now a READ-ONLY consumer of the LIVE production factory broker.**

`conf/things/mqtt.things` had every `commandTopic` deliberately stripped. Do not restore them.
The factory's openHAB 3 is still the real controller, and OH5's rules genuinely command hardware —
`on_startup.rules` fires `sendCommand(OFF)` at 39 tunnel lights + 39 button lights one minute after
every restart. With `commandTopic` present, restarting this dev box would switch real lamps at a
running factory.

The file carries a 20-line header comment restating this. Read it before editing.

---

## 2. What changed today

### 2.1 MQTT repointed to the live factory (committed + pushed, `1d25316`)

| Was | Now |
|---|---|
| `10.50.20.100:1883` (temporary Mosquitto LXC, **dead**) | `84.54.146.177:51883` (factory RPi4 over WAN) |
| `clientID="oh3srv"` | `clientID="oh5dev"` |
| 114 `stateTopic`+`commandTopic` pairs | `commandTopic` removed from all 114 |
| 19 flap `target_position` channels | commented out with `// [read-only mode]` |

Two hazards this avoided, both verified against the running factory:

1. **clientID collision.** `oh3srv` is the factory OH3's own client ID. MQTT brokers evict the older
   session on a duplicate ID — OH3 and OH5 would have kicked each other in a reconnect loop and the
   factory would have lost its one-minute control tick. OH3 was confirmed alive (~76 `cmnd/` messages
   every minute on the `:00` tick).
2. **OH5's rules command real hardware** (see §1).

Verified afterwards: live telemetry from all 18 tunnels reaches Items and persists to InfluxDB;
OH5 publishes **nothing** to `cmnd/`. Verification method — subscribe to `cmnd/#` on the broker and
correlate against OH5's `events.log` `ItemCommandEvent` lines. Traffic on `cmnd/` is expected; it is
OH3's. OH5 should log only `oneMinuteTriggerSwitch`, a virtual item with no MQTT channel.

### 2.2 Remote access to the production factory

The factory RPi4 (openHAB 3, openhabian, Raspbian **Buster**, 32-bit `armv7l`) now runs Tailscale:

- **`harvesta-oh3-rpi4` = `100.77.244.57`** — ports 8080 (openHAB/HABPanel), 3000 (Grafana),
  1883 (MQTT), 22 (SSH). Credentials in `Harvesta_Sredetc_Mosquitto_WAN_access_.txt`.

**Why Tailscale and not OpenVPN:** the factory LAN and this desktop's LAN are *both* `10.50.20.0/24`,
and the local one is a directly-connected route. A pushed `10.50.20.0/24` VPN route would have
hijacked the local openHAB LXC (.5), InfluxDB (.3) and the default gateway (.1) — a self-inflicted
outage on the working machine. Tailscale's `100.64.0.0/10` avoids the collision entirely.

Rules for that host:
- **Treat as read-only.** HTTP GETs and screenshots only. Never click a control or POST a command.
- **Never** `--advertise-routes=10.50.20.0/24` on it — that recreates the subnet clash.
- **Never** `apt-get upgrade` it. Buster is EOL, its main mirror 404s, and several repo keys are
  expired. That 404 is also what aborted the Tailscale install script before `apt-get install`;
  the fix was simply `sudo apt-get install -y tailscale` afterwards.

### 2.3 Operator handbook (the main deliverable)

**`Documents/Ръководство за оператора-v1.0.docx`** and `.pdf` — in the repo, committed.
22 pages, **Bulgarian only** (an earlier bilingual draft was rejected).

Build chain, in the desktop workspace:

| File | Role |
|---|---|
| `make_handbook.py` | **Entry point.** Two-pass build. Run this, not the one below. |
| `build_handbook.py` | Generates the .docx. Reads `toc_pages.json` if present. |
| `handbook_charts.py` | Three charts from real 2023 InfluxDB production data |
| `shoot_oh5.py` / `shoot_prod.py` | Playwright screenshots (OH5 / live factory OH3) |
| `handbook_demo_state.py` | Seeds or clears illustrative Item states on OH5 |
| `handbook_images/` | Images the document embeds |
| `handbook_out/` | Built .docx + .pdf, copied into the repo's `Documents/` |

Why two passes: a Word **TOC field** renders as "Натиснете F9…" until the reader refreshes it, and
headless PDF export never refreshes it — that shipped in the first draft. `make_handbook.py` now
renders once, reads back which page each heading landed on via `pdftotext`, and rebuilds with the
numbers as ordinary text.

Screenshot policy is a deliberate **hybrid**: production supplies КЛАПИ, ДИСПЛЕЙ, МАСТЕР РЕСЕТ and
ПЪРВОНАЧАЛЕН ПУСК; OH5 supplies the timer/temperature pages, because production is currently idle
and shows `−928` in red on every row, which teaches nothing about the colour coding. Both systems run
identical sitemaps, so layouts match.

---

### 2.4 Handbook §8 — reading the Grafana history (added later the same day)

The handbook gained a new **section 8, „Графики в Grafana — историята на тунелите"** (6 pages,
7 figures). Old §8/§9 became §9/§10; `make_handbook.py`'s `HEADINGS` list and the in-document TOC
were updated to match. Now **28 pages**, was 22.

Figures come from `shoot_grafana.py` (new). It is read-only in the same sense as `shoot_prod.py` —
plain GETs with the time range in the URL, `&theme=light` for print, `&kiosk=tv`, `viewPanel=` for
single panels. Every figure is taken from a dashboard the operators already have a desktop shortcut
for, so a reader can open the exact screen the book shows.

What the section teaches, and why each point is in there — all of it verified against InfluxDB
rather than read off the pictures, and all folded into `system_operation_knowledge.md`:

- Grafana **shows but does not control** — stated up front, because every other page in the book is
  a control page.
- Red = warm/`out`, blue = cold/`in`, on every dashboard. On humidity the order **inverts**.
- **Every edge of the button trace is one press**, rising or falling. The trace latches high for
  hours and an operator would otherwise read that as "the button is held down".
- **Notches = doors opened**; a **one-sample vertical needle is a sensor glitch** (the worked case
  is `in_1` reporting 38.5 °C between 66.6 and 66.8 °C).
- Two real overheats as the "what a problem looks like" pair: tunnel 6 at **89.3 °C** on the very
  day used as the normal example, and the 2021 burner-sensor failure at **101.9 °C** with the
  vertical drop where operators killed the burner by hand.

Three dashboards are recommended and the other seven explicitly told to be ignored — most are
abandoned drafts, and one (`XptktCZgk`) never got its button series added despite the title.

### 2.5 HABPanel colour coding — two gaps closed

Prompted by the question "does the handbook cover the HABPanel pages and their colour codes".
It covered ДИСПЛЕЙ but had two holes, both now filled in §5.6 and §6.3:

1. **The alarm tile collapses.** Reading `T-Rh-Timer-T-Alarm`'s `ng-if` conditions showed that a
   tunnel in temperature alarm stops rendering humidity, the timer and both cold-side values, and
   draws an amber triangle + red thermometer instead. The book previously described only the
   recolouring, which would leave an operator hunting for a timer that is deliberately hidden.
2. **УПРАВЛЕНИЕ КЛАПИ has no colour coding at all** — `-51`, `274`, `490` and `NULL` render in the
   same light blue as healthy values, on both the desktop and phone layouts. §6.4 asks operators to
   spot impossible values on a page that gives them no help; that is now said out loud.

The alarm screenshot is real, not mocked. The factory is off-season and the user confirmed a
temporary value change was fine, so `shoot_habpanel_alarm.py` briefly lowered
`default_tnl_temperature_alarm_value` **on OH5** below the live ambient readings (29.55 °C), put
tunnels 12–18 into alarm alongside 11 normal ones, screenshotted, and restored 86.0 in a `finally`.
Verified restored. Full method and reasoning in `system_operation_knowledge.md`.

> The OH5 `FlapControl-phone` view was captured too and **discarded** — its sliders all read `NaN`
> because read-only mode has the 19 `target_position` channels commented out. Any future OH5
> screenshot of a flap *target* will have the same artifact; use production for those.

Released as **v1.1** — `handbook_out/Ръководство за оператора-v1.1.docx` + `.pdf`, **29 pages**.
`OUTFILE` in `build_handbook.py`, `DOCX` in `make_handbook.py` and the title-page version string
were all bumped.

**Not committed.** The repo's `Documents/` still holds the 22-page v1.0, which is what operators
have in hand. Copy v1.1 in and commit once the content has been reviewed.

---

## 3. Domain corrections learned today

All folded into `system_operation_knowledge.md`. Several overturned earlier assumptions — do not
regress them.

| Topic | Correct understanding |
|---|---|
| **OUT (ТОПЪЛ) button** | Means **"not dry yet"** — the cart is rotated and pushed back for +30 min. It does *not* mean "cart being removed". |
| **IN (СТУДЕН) button** | The warm-side cart **was ready**: worker removes it, closes doors, goes to the cold side, opens doors, inserts a fresh cart, closes doors, presses. → new 120 min cycle. |
| **Flap scale 0–200** | Millimetres of **chain pull**. Full travel 200 mm = **20 cm**. (First described as 2 m — wrong.) The HABPanel `mm` labels are **correct**. |
| **Flap failure** | Potentiometer coupled **via a gear wheel** to the actuator. Symptoms: impossible value, or humidity not responding to the slider. Causes: potentiometer, jammed piston, seized flap, broken pull cable. |
| **Large negative timers on ALL tunnels** | **Normal** — system running but not yet in use. A large negative on a *single* tunnel is the real signal. Stop all tunnels at season start to zero them. |
| **Staggered filling** | The technologist's key judgement: hold each cart the right length of time so that when the 12th is in, every cart has reached the drying stage matching its position — as if the tunnel had been full all along. |

Rule-level behaviours worth remembering:
- **RESET / INIT switches never self-clear.** Auto-off timers are commented out; rules fire on
  `changed` + `state == ON`. Pressing an already-ON switch does nothing.
- **Timers run unbounded negative** — no floor in the decrement.
- **NULL enable state blanks the operator pages** — every widget is gated on `==ON` or `==OFF`.
- **`globalInit` rewrites every tunnel** including running ones; `set_all_to_defaults_sw` only
  touches disabled ones.
- **Colour codes differ per page.** ТЕКУЩИ ТАЙМЕРИ has *no* warning colour — green until it goes red.

---

## 3a. What the InfluxDB history actually contains

Surveyed 2026-08-10. Full detail in `data_inventory.md`.

| Season | Drying data |
|---|---|
| 2021 | ✅ — contains the burner-sensor incident (below) |
| 2022 | ✅ late Jul – Sep |
| 2023 | ✅ late Jul – mid Sep — **best season for analysis**; all handbook charts come from 28 Aug 2023 |
| 2024 | ✅ Aug – early Sep, coverage ends mid-Sep 2024 |
| **2025** | ❌ **no season at all** |
| 2026 | ⚠️ test replay (20 May – 8 Jun) + live data resumed 2026-08-10 |

**2025 is empty because the factory did not run** — climate change left no plum harvest, or a very
weak one. Confirmed by the operator. This is not data loss; do not go looking for it on the factory's
own InfluxDB.

**The 2021 burner-sensor incident** is why the high-temperature alarm exists. The gas burner's *own*
sensor failed; its independent controller therefore did not know the tunnel was overheating and kept
heating. Our sensors sit outside that control loop, so the rise stayed visible on the operator
screens — operators caught it and shut the burner down manually, saving tons of plums. It is written
up in handbook §7.2 as the justification for the 86 °C threshold.

Sustained episodes in the 2021 data (local, UTC+3): 05.09 21:38–22:19 tunnel 10 warm to **101.9 °C**
with tunnels 10 cold and 11 warm going high in the same window; 15.09 17:36–18:37 tunnel 3 warm to
94.5 °C; 01.09 tunnels 7 and 4. The 05.09 cluster — three sensors, two adjacent tunnels, same window
— points at the heating rather than one bad sensor.

> **Querying caveat:** filter `value < 500`. `in_5` and `in_12` report **998** as a sensor error
> code, and isolated two-minute spikes to 121.6/121.7 °C are glitches, not real temperatures.
> Which specific date the operator remembers as *the* incident is **not confirmed** — 05.09 and
> 15.09.2021 are both candidates.

---

## 4. Environment gotchas discovered

- **openHAB does not hot-reload config written through the mount.** A server-side `touch` doesn't
  help either. Use `ssh openhab 'systemctl restart openhab'`.
- **`conf/` files are CRLF.** Preserve line endings when scripting edits or the whole file diffs.
- **The mount is CIFS** (`//100.83.124.116/oh5harvrepo`). Existing files are writable; **creating
  directories or new files fails with Permission denied.** Do it server-side and `scp` the content.
- **This desktop has no sshd and no SMB/NFS share** — files here are not reachable from Windows.
- **No `node`/`npm`** (only Playwright's bundled node, without the `docx` module). The handbook is
  built with `python-docx`.
- **OOXML enforces child-element order.** `build_handbook.py` has an `insert_ordered()` helper; a
  blind `append` produces files LibreOffice tolerates but that fail schema validation.
- The harness **blocks inline SSH passwords** on the command line. `ssh openhab` is key-based; for
  the factory Pi read the credential from the file rather than typing it into a command.

---

## 5. Open items

**Flap hardware (real, in progress).** 7 of 19 flaps report impossible positions — flap 4 = −51,
18 = −5, 7 = 274, 10 = 490, and 8/16/19 NULL. Operator confirmed: **broken potentiometers, under
repair**. Used as the worked example in handbook §6.4.

**Deferred from the code review** (unchanged): legacy Xtend DSL, extreme rule duplication (108
copy-pasted blocks), stub `temperature_alarm.rules`, plaintext credentials.

**Security worth raising.** `84.54.146.177:51883` is internet-facing, unencrypted (`secure=false`),
Mosquitto **1.5.7** (2019), with the password in a committed file. Acceptable for a read-only
observer; less so once OH5 is the thing opening and closing tunnels. Now that Tailscale reaches the
factory, MQTT could move onto the tailnet and that public port could close.

**Runtime state — untracked, but still on the server** (done 2026-08-10, `ae14745`). These are
`.gitignore`d now; read them directly via `ssh openhab` when needed:
- `userdata/jsondb/users.json` — accounts + 18 live OAuth refresh tokens, an authorization code, a
  pendingToken. Changes on every login.
- `userdata/config/.../WatchService/*.config` — per-boot UUID file, pure noise.

`userdata/config/org/openhab/addons.config` is deliberately **kept tracked** — it records the
installed addons and is genuine config; only its felix revision counter churns.

**Secrets still in the repo, unresolved.** Untracking would not help, because they are already in the
pushed history — only rotation fixes them:
- `userdata/secrets/rsa_json_web_key.json` — the private RSA key openHAB signs tokens with
- `userdata/openhabcloud/secret` — the openHAB Cloud pairing secret
- plaintext credentials in `conf/things/mqtt.things` and `conf/services/influxdb.cfg`

Accepted so far because the repo is private.

**Handbook gaps.** ~~Grafana dashboards are not covered.~~ **Done** — see §7 below. Still open: the
`Линкове към страниците за управление` link points at `sitemap=masterinit`, which **does not exist**
on either OH5 or production — a dead link that needs rebuilding. All 18 operator links still point at
`10.50.20.100` and will need reissuing at switchover.

**Grafana follow-ups** (raised while writing §7, none of them done):
- The **local Grafana at `10.50.20.4` renders every dashboard empty** — its `openhab_home` datasource
  points at `http://10.50.20.4:8086`, where nothing listens. Repoint it at `10.50.20.3:8086`.
- The operator shortcut folder has **seven** Grafana links, **four of them to obsolete dashboards**,
  and none to `sOvH8URRk`. Worth pruning when the links are reissued at switchover.
- `in_1_button`'s label in `controller.items` reads `"Тунел 1 LED"`, copy-pasted from
  `in_1_tnl_light`. Cosmetic, but it misdescribes the channel.

---

## 6. The next milestone

**Production switchover** — when this OH5 server replaces the factory RPi4 as the live controller:

1. **Stop openHAB 3 on the factory RPi4 first.** Two controllers on one broker will fight over every
   relay. Confirm it is off the bus.
2. Re-add `commandTopic` to all 114 channel pairs (`cmnd/<device>/POWER<n>` mirroring each
   `stat/<device>/POWER<n>`).
3. Un-comment the 19 flap `target_position` channels (marked `// [read-only mode]`).
4. Keep `clientID="oh5dev"` unless OH3 is provably gone.
5. Restart openHAB and verify OH5 now owns the `cmnd/` traffic.

Restore reference: git history of `conf/things/mqtt.things` **before commit `1d25316`**.
