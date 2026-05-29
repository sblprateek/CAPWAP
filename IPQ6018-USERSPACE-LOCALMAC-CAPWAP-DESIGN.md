# IPQ6018 Userspace Local-MAC CAPWAP WTP — Design

**Target:** Qualcomm IPQ6018 AP, QSDK 12.2, proprietary **qca-wifi** (full-offload) radio.
**Stack:** Travelping CAPWAP — `freewtp` (WTP) ↔ `capwap` (Erlang AC) + `capwap-dp` (AC datapath). All three are one matched vendor stack (github.com/travelping/{freewtp,capwap,capwap-dp}).
**Goal:** a working, deployable CAPWAP WTP **in 10–15 days** without a kernel patch, without NSS, without switching to ath11k.
**Status of claims:** items marked ✅ are verified against the source in this tree (file:line cited); items marked ⟶ are proposed changes.

---

## 1. Core idea

qca-wifi is full-offload — it owns the entire 802.11 MAC/MLME/CCMP in firmware and *insists* on terminating Wi-Fi itself. So we do **not** try Split-MAC (which needs raw 802.11 at the controller) and we do **not** use freeWTP's kernel data path (which taps mac80211 — absent here). Instead:

- qca-wifi + hostapd terminate Wi-Fi and expose client traffic as **802.3 Ethernet** on the VAP netdev.
- A small **userspace forwarder** tunnels that 802.3 to the AC as **CAPWAP Local-MAC / 802.3** data.
- **freeWTP** runs only the **CAPWAP control plane** and hands the forwarder the negotiated session.

This is CAPWAP **Local-MAC**, which is freeWTP's default (`freewtp/src/wtp.c:62` `CAPWAP_LOCALMAC`) ✅ and the AC's default (`capwap/src/capwap_config_env.erl:77` `mac_mode = local_mac`) ✅.

---

## 2. Components and roles

| Component | Role | State |
|---|---|---|
| **qca-wifi + hostapd** | Owns radio: 802.11 MAC, MLME, association, CCMP. Exposes client traffic as 802.3 on the VAP (`ath0`). | Existing on board |
| **freeWTP** (control) | CAPWAP FSM (Discovery→Join→Configure→DataCheck→Run) + control DTLS; negotiates Local-MAC; holds session. | Existing; **bounded change** (§7) |
| **wtp-dp** (data, NEW) | 802.3 ⇄ CAPWAP data tunnel: encap/decap, fragmentation, data keepalive. | **New**, adapted from `capwap-dp` |
| **capwap + capwap-dp** (AC) | Controller + AC datapath. | Existing; **config only** |

---

## 3. Architecture

```
        ┌──────────────────────── IPQ6018 AP ────────────────────────┐
 client │   ┌────────────┐   802.3    ┌─────────────────────┐  CAPWAP │
 ~~~~~~~~~~► │ qca-wifi   │ ─────────► │  wtp-dp (NEW)       │  data   │
 802.11 │   │ + hostapd  │ ◄───────── │  AF_PACKET/TAP ⇄UDP │  /UDP   │
        │   │ owns 802.11│   802.3    │  encap/frag/keepliv │  :5247  │
        │   └─────┬──────┘            └──────────┬──────────┘         │
        │         │ assoc events                 ▲ session params     │
        │         ▼                              │ (peer,sid,port,mtu)│
        │   ┌───────────────────────────────────┴────────┐  CAPWAP   │
        │   │ freeWTP (control only): FSM + DTLS,          │  control  │
        │   │ Local-MAC, presents radio, gets WLAN config  │  /UDP     │
        │   └────────────────────────────┬─────────────────┘ :5246    │
        └────────────────────────────────┼───────────────────────────┘
                                          ▼
                          ┌───────────────────────────────┐
                          │ Travelping AC: capwap (Erlang) │
                          │ control + capwap-dp datapath   │
                          └───────────────────────────────┘
```

---

## 4. Data plane (wtp-dp)

**Uplink (client → AC)**
1. Client → 802.11; qca-wifi decrypts + converts to **802.3**, delivers on `ath0`.
2. wtp-dp captures the Ethernet frame (AF_PACKET raw socket on `ath0`, or `ath0` in a bridge/TAP).
3. Prepend CAPWAP **data** header (802.3 payload), fragment if > tunnel MTU. *Inverse of `capwap-dp/src/worker.c:637+` reassembly and `capwap-dp/src/capwap-dp.c:406` frag check.* ✅ (reference)
4. `sendto()` over UDP to AC data port (DTLS optional — deferred).

**Downlink (AC → client)**
1. `recvfrom()` CAPWAP data; reassemble; strip header → 802.3 frame.
2. Inject onto `ath0`.
3. qca-wifi does 802.11 MAC + CCMP → client.

**Data keepalive:** wtp-dp sends/answers the CAPWAP data-channel keepalive that binds the data tunnel to the control session (what the old kmod's `wtp_kmod_send_keepalive` did, `freewtp/src/kmod.h:58`) ✅.

The forwarder handles `CAPWAP_802_3_PAYLOAD` (`capwap-dp/src/worker.c:535`) ✅; the `CAPWAP_802_11_PAYLOAD` path (`worker.c:540`) is left unused (Split-MAC, not our mode).

---

## 5. Control plane (freeWTP)

freeWTP runs its existing FSM + control DTLS, advertising **WTP MAC Type = Local-MAC** (`freewtp/lib/element_wtpmactype.c`) ✅ and **802.3 frame tunnel** (`freewtp/src/wtp.c:958` `CAPWAP_WTP_8023_FRAME_TUNNEL`) ✅. It completes Join/Configure and receives the WLAN/SSID/security config from the AC.

Two outputs from the session:
- **WLAN config → hostapd.** freeWTP isn't driving the radio, so AC-dictated SSID/security is applied to **hostapd**. First cut: statically match hostapd to the AC's WLAN; later: auto-apply.
- **Tunnel params → wtp-dp.** freeWTP hands the forwarder peer address, **session-id**, data port, **MTU** — the arguments of `wtp_kmod_create(family, peeraddr, sessionid, mtu)` (`freewtp/src/kmod.h:53`) ✅.

**Station reporting (Local-MAC):** association events originate in **hostapd**, not freeWTP. First cut: `capwap-dp`'s station table learns client MACs from tunneled 802.3 source addresses (`capwap-dp/src/worker.c:254 find_station`) ✅. Follow-on: a hostapd→freeWTP event bridge emitting proper CAPWAP Station-Config messages.

---

## 6. freeWTP ↔ wtp-dp control socket (new internal API)

A tiny local IPC (UNIX domain socket) replaces the genl-to-kmod boundary. It mirrors the existing `kmod.h` API 1:1, so freeWTP's call sites don't change — only the implementation behind them:

| Message (freeWTP → wtp-dp) | Mirrors `kmod.h` | Payload |
|---|---|---|
| `CREATE` | `wtp_kmod_create` (`:53`) | family, peer sockaddr, session-id, MTU, flags(8023) |
| `RESET` | `wtp_kmod_resetsession` (`:55`) | — |
| `ADD_STATION` | `wtp_kmod_add_station` (`:69`) | radioid, MAC, wlanid, flags |
| `DEL_STATION` | `wtp_kmod_del_station` (`:70`) | radioid, MAC |
| (data keepalive owned by wtp-dp) | `wtp_kmod_send_keepalive` (`:58`) | — |

Data frames are **not** sent over this socket (unlike the old kmod's `send_data`) — wtp-dp moves data directly between `ath0` and the tunnel for performance.

---

## 7. The freeWTP change (the crux, and the schedule risk)

**Verified problem:** freeWTP gates its *entire* data-plane wiring **and** `wifi_driver_init()` on `binding == IEEE80211`:
- `freewtp/src/wtp.c:55` default `binding = CAPWAP_WIRELESS_BINDING_NONE` ✅
- `freewtp/src/wtp.c:926-933` binding switch: `IEEE80211` → `wifi_driver_init()` ✅
- `freewtp/src/wtp.c:1044, 1538` data-plane/teardown gated on `IEEE80211` ✅

So `binding=NONE` ⟶ no data plane and no radio advertised (AC Configure has no target); `binding=IEEE80211` ⟶ data plane wired **but** freeWTP drives the radio via nl80211 → dual-MLME conflict with qca-wifi+hostapd.

**The change (proposed):** decouple "present a radio + run the data-plane boundary" from "drive a real nl80211 radio."
1. ⟶ Add a **null-radio / external-datapath mode** so freeWTP presents a radio to the AC (Join/Configure work) while `wifi_driver_init()` is a no-op shim (no nl80211 driving → no MLME fight).
2. ⟶ Re-implement the `src/kmod.c` boundary to speak the §6 socket to **wtp-dp** instead of genl to the (unusable) kernel module.

Bounded (lives in `wtp.c` binding init + `kmod.c`), but it is source work and the most likely item to consume schedule — hence 10–15, not 5–10.

---

## 8. Code changes by repository

### 8.1 `freewtp/` — the real engineering

| File | Change | Notes |
|---|---|---|
| `src/wtp.c` | ⟶ Add null-radio/external-datapath binding mode. Touch points: default `:55`, `:62-63`; binding parse `:915`; binding init switch `:926-933`; teardown `:1538`; data-plane gate `:1044`. | Present a radio without `wifi_driver_init()` driving the real one. |
| `src/kmod.c` | ⟶ Re-target the data-plane API to the §6 UNIX socket. Keep `src/kmod.h` API unchanged (`create/reset/keepalive/add_station/del_station`). | `send_data` becomes unused (wtp-dp moves data directly). |
| `src/binding/ieee80211/` | ⟶ Optional: a thin "null binding" that satisfies the FSM's radio/WLAN callbacks without nl80211. | Alternative to the no-op `wifi_driver_init`. |
| `etc/wtp.conf` | ⟶ Set `binding`/new `datapath = "external"`; `mactype = "localmac"` (already `:20`); `tunnelmode { ethframe = true; }` (already `:16`). | Config keys mostly exist already ✅. |
| `kmod/`, `kernel-patches/` | ⟶ **Drop from the build** (mac80211-only; irrelevant to qca-wifi). | Confirmed unusable: `kmod/netlinkapp.c:607` registers via `ieee80211_pcktunnel_register`; even 802.3 mode captures from mac80211 first (`kmod/netlinkapp.c:115`) ✅. |
| `Makefile.am` / `openwrt/` recipe | ⟶ Stop building `kmod/`; add `wtp-dp` (or package separately). | — |

### 8.2 `wtp-dp/` — NEW forwarder (small)

A new standalone userspace daemon. Borrow the proven CAPWAP datapath code from `capwap-dp`, strip its Erlang coupling, drive it from freeWTP's §6 socket.

| Source | Origin | Change |
|---|---|---|
| CAPWAP header parse/build, fragmentation/reassembly | port from `capwap-dp/src/worker.c` (`:637+`) + `capwap-dp/include/` | Reuse as-is (both directions). |
| Station table (MAC → tunnel) | port from `capwap-dp/src/worker.c` (`:166-265`) | Reuse for downlink demux. |
| Local capture/inject | NEW | AF_PACKET raw socket on `ath0` (or TAP in a bridge). |
| Control endpoint | NEW (replaces `capwap-dp.c` `ei_cnode`/`erlang_pid` at `:98-105`) | Listen on the §6 UNIX socket from freeWTP; **remove** the Erlang `erl_interface` dependency. |
| Event loop | keep `libev` model from `capwap-dp` | — |

### 8.3 `qca-hostap/` (hostapd) — config, not source (first cut)

| Item | Change |
|---|---|
| `hostapd.conf` for the VAP | ⟶ Bring up the BSS (SSID/security) matching what the AC pushes. Static for first cut. |
| Association-event bridge (follow-on) | ⟶ Small external script using hostapd's `ctrl_iface`/event monitor → feed freeWTP `ADD_STATION/DEL_STATION` (§6). No hostapd source change. |

### 8.4 `capwap/` + `capwap-dp/` (AC) — config only

| Item | Change |
|---|---|
| `capwap` AC config | ⟶ `mac_mode = local_mac` (already default `capwap_config_env.erl:77` ✅); register the WTP certificate (see `capwap/docs/certificates.md`); set data/control ports. |
| `capwap-dp` | ⟶ Run as-is on the AC side; no source change. |

### 8.5 QSDK packaging

| Item | Change |
|---|---|
| Image config | ⟶ Keep `qca-wifi` + `qca-hostap`; add `freewtp` (sans kmod) + `wtp-dp` packages. No `kmod-*` for CAPWAP, no NSS capwapmgr, no ath11k. |

---

## 9. Reused / new / changed / deleted

| | |
|---|---|
| **Reused as-is** | freeWTP control FSM + control DTLS; CAPWAP message elements (incl. WTP MAC Type); `capwap-dp` framing/frag/station code; qca-wifi + hostapd. |
| **New (small)** | `wtp-dp` forwarder; freeWTP↔wtp-dp UNIX socket (§6). |
| **Changed (bounded)** | freeWTP null-radio/external-datapath mode (`wtp.c`) + `kmod.c` re-target. |
| **Deleted / unused** | freeWTP `kmod/` + `kernel-patches/`; the kmod `send_data` path. |

---

## 10. Milestones (10–15 days)

1. **Days 1–2 — Control smoke test (go/no-go).** Build freeWTP, run vs the Travelping AC, confirm Discovery→Join→Configure in `local_mac`. Retires the biggest unknown before datapath code.
2. **Days 3–6 — freeWTP change (§7).** Null-radio/external-datapath mode; `kmod.c` → §6 socket; freeWTP boots, joins, and *emits* session params instead of touching the kernel.
3. **Days 6–10 — wtp-dp.** Adapt `capwap-dp/worker.c` for the WTP direction: AF_PACKET on `ath0` ⇄ CAPWAP-802.3 UDP, encap/decap, fragmentation, data keepalive.
4. **Days 10–13 — End-to-end.** hostapd BSS up; real client associates; traffic reaches AC over the tunnel and back; MTU/fragmentation correctness.
5. **Days 13–15 — Stabilize.** Reconnect/keepalive recovery, basic soak, config cleanup. Buffer.

---

## 11. Risks & mitigations

| Risk | Mitigation / evidence |
|---|---|
| freeWTP↔AC interop | Matched Travelping stack (same vendor, verified §0) → low; smoke-tested day 1. |
| Dual-MLME with qca-wifi | Null-radio mode: freeWTP never drives the radio; hostapd owns it (Local-MAC). §7 |
| Data binding mismatch | Verified: freeWTP defaults Local-MAC/802.3 (`wtp.c:62`, `:958`), AC defaults `local_mac`. |
| Datapath from scratch | Not from scratch — adapt `capwap-dp` (same vendor, same framing). §8.2 |
| Station reporting | First cut: AC learns MACs from data plane (`worker.c:254`); hostapd→freeWTP bridge later. |
| Data-channel DTLS | Deferred; CAPWAP allows plaintext data channel for first cut. |

---

## 12. What it delivers — and its limits

**Delivers:** a working, deployable Local-MAC CAPWAP WTP on the existing qca-wifi board, interoperating with the Travelping AC, with centralized control + AC-as-data-anchor — no kernel patch, no NSS, no ath11k.

**Limits (by design):** throughput is **userspace-limited** (per-packet copy + userspace framing on the A53s — fine for modest client counts, not line-rate Wi-Fi-6); data-channel DTLS and proper Station-Config messaging are follow-ons; the Local-MAC trade-offs apply (no controller-side raw-802.11 features — which don't fit this silicon anyway).

**Upgrade paths (same control plane):**
- Higher throughput, no controller data-anchor → **local bridging** (`wtp.c:964` `CAPWAP_WTP_LOCAL_BRIDGING`).
- Hardware offload → the **NSS capwapmgr** design in `IPQ6018-NSS-CAPWAP-ARCHITECTURE.md` (months, but line-rate).

---

*Generated from source inspection of this QSDK 12.2 tree and the Travelping freewtp/capwap/capwap-dp repos. ✅ = verified against cited file:line; ⟶ = proposed change.*
