# Handoff — Harvesta openHAB, session of 2026-08-10

Written for a fresh agent session with no memory of this work. Read `CLAUDE.md` first for the
standing project rules; this file covers **what changed today and what is still open**.

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

**Uncommitted runtime churn in the repo.** `userdata/jsondb/users.json` (contains live OAuth
authorization codes and a pendingToken), `userdata/config/.../addons.config`, and a WatchService UUID
file. Deliberately **not committed** — pushing `users.json` would publish working session
credentials. Worth `.gitignore`-ing.

**Handbook gaps.** Grafana dashboards (six of them, `:3000`) are not covered. The
`Линкове към страниците за управление` link points at `sitemap=masterinit`, which **does not exist**
on either OH5 or production — a dead link that needs rebuilding. All 18 operator links still point at
`10.50.20.100` and will need reissuing at switchover.

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
