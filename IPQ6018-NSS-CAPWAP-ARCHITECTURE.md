# freeWTP on IPQ6018 (QSDK 12.2) — Architecture Design

**Target:** Qualcomm IPQ6018 AP, QSDK 12.2, proprietary `qca-wifi` radio driver (no mac80211).
**Goal:** Run a CAPWAP WTP without writing one from scratch — reuse freeWTP's CAPWAP control brain, and replace its mac80211-coupled data plane with the NSS CAPWAP hardware offload the SDK already ships.

Every claim below is backed by a file:line citation in the **Proofs** section. Paths are relative to the repo root (`freewtp/…` and `qsdk/…`).

---

## 1. Problem statement

freeWTP's data plane is built for **mac80211 SoftMAC** drivers. It needs:

- a kernel module `smartcapwap_wtp` (`freewtp/kmod/`) that tunnels raw 802.11 frames over CAPWAP/UDP, and
- a **mac80211 kernel patch** (`freewtp/kernel-patches/mac80211_packet_tunnel-*.patch`) that adds RX/TX taps (`ieee80211_pcktunnel_register`, `ieee80211_inject_xmit`) inside `net/mac80211/{rx,tx,iface}.c`.

The IPQ6018 radio runs `qca-wifi` (firmware/full-offload). **mac80211 is not in its data path**, so there is nothing for that patch to hook. Porting the patch into the proprietary datapath would be large, fragile, and would defeat NSS hardware offload.

**Key realization:** the function that patch+kmod provide (CAPWAP encap/decap/fragmentation/keepalive of wireless frames) is already implemented in dedicated hardware by the **NSS CAPWAP manager** that ships in this SDK. So we *delete* the patch+kmod and re-point freeWTP at NSS.

---

## 2. Architecture overview

```
            ┌──────────────────────────────────────────────────────────────┐
            │                        USER SPACE                             │
            │                                                               │
            │   ┌───────────────────┐        ┌──────────────────────────┐   │
   AC  <════╪══▶│  freeWTP daemon   │        │      qca-hostapd         │   │
 (CAPWAP    │   │  (control plane)  │        │  (owns 802.11 MLME on    │   │
  control   │   │  - DTLS handshake │        │   qca-wifi: beacon,      │   │
  channel)  │   │  - Discovery/Join │        │   auth/assoc, keys, STA) │   │
            │   │  - Configure FSM  │        └────────────┬─────────────┘   │
            │   │  - Tunnel orchestr│                     │ nl80211/cfg80211 │
            │   └─────────┬─────────┘                     │                  │
            │             │ genl: "smartcapwap_wtp"       │                  │
            └─────────────┼───────────────────────────────┼──────────────────┘
                          │                               │
            ┌─────────────▼───────────────────┐  ┌────────▼──────────────────┐
            │  capwap_nss_shim.ko (NEW)        │  │   qca-wifi (VAP: ath0)    │
            │  registers "smartcapwap_wtp"     │  │   osif_dev.nss_ifnum  ────┼──┐
            │  genl family, translates to:     │  └───────────────────────────┘  │
            │  nss_capwapmgr_* + capwap0 netdev│                                  │
            └─────────────┬────────────────────┘                                 │
                          │ in-kernel API + capwap0 netdev                        │
            ┌─────────────▼───────────────────────────────────────────────┐      │
   AC  <════╪══▶│      NSS CAPWAP offload (qca-nss-clients/capwapmgr)       │◀─────┘
 (CAPWAP    │   │  HW encap/decap/frag/keepalive + DTLS offload (firmware)  │  update_src_interface
  DATA      │   └──────────────────────────────────────────────────────────┘  binds VAP node→tunnel
  channel)  │                       KERNEL / NSS FIRMWARE
            └───────────────────────────────────────────────────────────────┘
```

Three planes:

1. **CAPWAP control plane** — freeWTP daemon, unchanged. DTLS + Discovery/Join/Configure/Run FSM with the AC.
2. **Radio / 802.11 MLME** — qca-hostapd, which already speaks qca-wifi correctly. freeWTP does *not* drive the radio MLME directly (see §6 for why).
3. **CAPWAP data plane** — NSS CAPWAP offload, driven by a thin new shim that re-uses freeWTP's existing kernel interface.

CAPWAP binding model: **Local-MAC** (`NSS_CAPWAP_PKT_TYPE_802_3`). This matches NSS offload and avoids dual-MLME conflict.

---

## 3. What is deleted, kept, added

| freeWTP component | Action | Reason |
|---|---|---|
| `src/dfa_*.c`, `wtp.c` (control FSM + DTLS) | **Keep** | chipset-agnostic |
| `kmod/` (smartcapwap module) | **Delete** | NSS does this in HW |
| `kernel-patches/mac80211_*.patch` | **Delete** | mac80211 not in qca-wifi datapath |
| `src/kmod.c` boundary (9 calls) | **Re-point** | now drives the shim / NSS |
| `src/binding/ieee80211/wifi_nl80211.c` | **Demote** | radio owned by qca-hostapd, not freeWTP |
| `capwap_nss_shim.ko` | **Add (new)** | translate `smartcapwap_wtp` genl → `nss_capwapmgr_*` |

---

## 4. Data-plane design (the core substitution)

### 4.1 Tunnel lifecycle — freeWTP call → NSS API

| freeWTP `src/kmod.h` call | NSS capwapmgr equivalent |
|---|---|
| `wtp_kmod_link` | `nss_capwapmgr_netdev_create()` → get `capwap0` once |
| `wtp_kmod_create(family,peer,sessionid,mtu)` | `nss_capwapmgr_ipv4_tunnel_create(dev,tid,&ip_rule,&capwap_rule,dtls)` then `nss_capwapmgr_enable_tunnel()` |
| `wtp_kmod_resetsession` | `nss_capwapmgr_disable_tunnel()` + `nss_capwapmgr_tunnel_destroy()` |
| `wtp_kmod_send_keepalive` / `RECV_KEEPALIVE` | firmware-owned; poll `nss_capwapmgr_tunnel_stats()` |
| `wtp_kmod_send_data` / `RECV_DATA` | fast-path; only daemon-originated frames use the metaheader TX path (§4.2) |
| `wtp_kmod_join_mac80211_device(wlan,flags)` | `nss_capwapmgr_update_src_interface(capwap0,tid,vap_nss_ifnum)` — see §5 |
| `wtp_kmod_leave_mac80211_device` | unbind / destroy tunnel |
| `wtp_kmod_add_station` / `del_station` | qca-wifi already tracks STAs; optional `nss_capwapmgr_add_flow_rule()` / `del_flow_rule()` |
| DTLS (was userspace wolfSSL) | optional offload: `nss_capwapmgr_configure_dtls(dev,tid,enable,&nss_dtlsmgr_config)` |

`nss_capwap_rule_msg.encap` carries exactly what `wtp_kmod_create` supplies: `src_ip/src_port`, `dest_ip/dest_port` (5246), `path_mtu`.

### 4.2 Per-packet contract (only for frames the daemon must originate/receive)

- **TX (WTP→AC):** build skb = `nss_capwap_metaheader{ tunnel_id, rid, type=802_3|802_11|WIRELESS_INFO, nwireless, wl_info }` + payload, reserve `NSS_CAPWAP_HEADROOM` (256), `dev_queue_xmit(capwap0)`. Firmware adds CAPWAP/UDP/IP/eth (+DTLS).
- **RX (AC→WTP):** NSS decaps and `netif_rx`'s onto `capwap0` with the metaheader prepended.

In Local-MAC steady state these per-packet paths are **not** used for STA traffic — frames flow VAP↔tunnel entirely in firmware (§5). They exist only for control/exception frames.

---

## 5. The bridge that replaces the kernel patch (VAP ↔ tunnel)

The mac80211 patch existed only to tap frames at the radio. On qca-wifi this becomes an **NSS node interconnect** — no patching:

1. Each qca-wifi VAP netdev's private struct `osif_dev` holds `nss_if_num_t nss_ifnum` — the VAP's NSS firmware node, allocated at VAP creation via `nss_dynamic_interface_alloc_node(NSS_DYNAMIC_INTERFACE_TYPE_VAP)`.
2. The shim's JOIN handler resolves the VAP netdev from freeWTP's `wlan->virtindex`, reads `osifp->nss_ifnum`, and calls `nss_capwapmgr_update_src_interface(capwap0, tid, nss_ifnum)`.
3. That rebuilds the tunnel's NSS IP rule with `src_interface_num = VAP node`, wiring **VAP → CAPWAP encap** in the fast path (uplink).

```c
struct net_device *vap = dev_get_by_index(&init_net, virtindex);
osif_dev *osifp = netdev_priv(vap);          /* qca-wifi VAP priv */
nss_capwapmgr_update_src_interface(capwap0, tid, osifp->nss_ifnum);
dev_put(vap);
```

**Downlink** (AC→tunnel→VAP→STA): bind the CAPWAP decap node's next-hop to the VAP node. For Local-MAC the simplest, proven route is to **bridge the VAP netdev and `capwap0`** and let standard NSS bridge/flow offload move both directions; the explicit next-hop API (cf. `nss_trustsec_tx_update_nexthop`, and the `wifi_meshmgr` vdev↔node pattern) is the alternative if a bridge is undesirable.

---

## 6. Control-plane design (why qca-hostapd, not freeWTP, owns the radio)

freeWTP's `wifi_nl80211.c` drives the radio like hostapd-on-mac80211: it creates an AP vif, pushes the full beacon, and **registers for management frames to run the entire 802.11 MLME in userspace** (`NEW_INTERFACE`→`START_AP`/`SET_BEACON`/`SET_BSS`→`REGISTER_FRAME`→`NEW_STATION`/`SET_KEY`).

qca-wifi supports all those cfg80211 ops, **but it also runs the MLME itself in umac/firmware** and forwards Auth/Assoc/Deauth/Action up to userspace only to supplement WPA/IE/WPS handling. Two userspace MLMEs (freeWTP + driver) would conflict.

**Decision:** qca-hostapd owns the radio (it is purpose-built for this driver). freeWTP runs as the CAPWAP control/orchestration daemon: it negotiates with the AC, and applies AC-dictated BSS config through the same config surface qca-hostapd uses, then sets up the NSS tunnel (§4) and binds the VAP (§5). This is the **CAPWAP Local-MAC** model and is the natural fit for offloaded silicon. Split-MAC (freeWTP owning raw 802.11 via a qca-wifi raw/monitor VAP) is possible but not worth the cost on this chipset.

---

## 7. Proofs (evidence in the trees)

### A. NSS CAPWAP offload exists and is the data-plane replacement
- `qsdk/qca/src/qca-nss-clients/capwapmgr/nss_capwapmgr.c` — the manager module (`obj-m += qca-nss-capwapmgr.o`).
- API surface — `qsdk/qca/src/qca-nss-clients/exports/nss_capwapmgr.h`: `nss_capwapmgr_netdev_create` (146), `nss_capwapmgr_ipv4_tunnel_create` (158), `nss_capwapmgr_enable_tunnel` (184), `nss_capwapmgr_update_path_mtu` (205), `nss_capwapmgr_update_src_interface` (227), `nss_capwapmgr_configure_dtls` (284), `nss_capwapmgr_tunnel_destroy` (341), `nss_capwapmgr_add_flow_rule` (358), `nss_capwapmgr_tunnel_stats` (400).
- Rule shape matches freeWTP `create` args — `qsdk/qca/src/qca-nss-drv/exports/nss_capwap.h:230` `nss_capwap_rule_msg`; `nss_capwap_encap_rule` = src/dst IP+port + path_mtu.
- Frame types incl. raw 802.11 and 802.3 — `qsdk/qca/src/qca-nss-clients/exports/nss_capwap_user.h`: `NSS_CAPWAP_PKT_TYPE_802_11`, `_802_3`, `_WIRELESS_INFO`, `_DTLS_ENABLED`; `nss_capwap_metaheader` carries `rid`, `tunnel_id`, `wireless_qos`, `wl_info`.
- Data-path contract — `nss_capwapmgr.c:244` `nss_capwapmgr_start_xmit` (TX prepends metaheader, `nss_capwap_tx_buf`, requires `NSS_CAPWAP_HEADROOM`); `nss_capwapmgr.c:3249` `nss_capwapmgr_receive_pkt` (RX with metaheader). `NSS_CAPWAP_HEADROOM 256` at `nss_capwap.h:36`.

### B. VAP↔tunnel binding is a supported node hookup (no driver patch)
- `qsdk/qca/src/qca-wifi/os/linux/src/osif_private.h:381` — `nss_if_num_t nss_ifnum;` in `osif_dev`.
- `qsdk/qca/src/qca-nss-drv/exports/nss_cmn.h:33` — `typedef int32_t nss_if_num_t;`.
- `qsdk/qca/src/qca-wifi/os/linux/src/osif_nss_wifiol_vdev_if.c:1274` — `nss_dynamic_interface_alloc_node(NSS_DYNAMIC_INTERFACE_TYPE_VAP)` assigns it.
- `qsdk/qca/src/qca-nss-clients/capwapmgr/nss_capwapmgr.c:1535` — `nss_capwapmgr_update_src_interface` writes `ip_rule.v4.src_interface_num = src_interface_num` and rebuilds the rule.

### C. The control-path facts behind the Local-MAC decision
- qca-wifi cfg80211 AP ops present — `qsdk/qca/src/qca-wifi/os/linux/src/ieee80211_cfg80211.c:5898+`: `.add_virtual_intf`, `.start_ap`, `.change_beacon`, `.stop_ap`, `.change_bss`, `.add_key`, `.add_station`/`.change_station`/`.del_station`, `.mgmt_tx`.
- qca-wifi forwards mgmt to userspace but also runs MLME — `qsdk/qca/src/qca-wifi/os/linux/src/osif_umac.c:5385-5428` (`cfg80211_rx_mgmt` for Auth/Assoc/Reassoc/Disassoc/Deauth/Action, gated on `vap->iv_cfg80211_create`).
- freeWTP expects full userspace MLME — `freewtp/src/binding/ieee80211/wifi_nl80211.c`: `nl80211_wlan_create` (NEW_INTERFACE), `nl80211_wlan_setbeacon` (START_AP/SET_BEACON/SET_BSS), `nl80211_wlan_registerframe` (REGISTER_FRAME), `nl80211_station_authorize` (NEW_STATION).
- qca-hostapd present — `qsdk/qca/src/qca-hostap/`.

### D. This chipset is supported, and the shim pattern already exists in-tree
- DTLS offload explicitly for IPQ60xx — `qsdk/qca/src/qca-nss-clients/Makefile` selects `DTLSMGR_DIR:=v2.0` commented *"DTLS Manager v2.0 for IPQ807x/IPQ60xx/IPQ50xx"*; `dtls/v2.0/` present.
- **Reference for the shim:** `qsdk/qca/src/qca-nss-clients/netlink/nss_nlcapwap.c` is a generic-netlink family (`NSS_NLCAPWAP_FAMILY`, `genl_ops` at lines 188-219) whose `.doit` handlers (`create_tun`, `destroy_tun`, `update_mtu`, `dtls`, `meta_header`, `tx_packets`, `keepalive`, …) drive the very `nss_capwapmgr_*` API above. This *proves* the proposed "genl family → capwapmgr" translation is a pattern Qualcomm already ships and builds — `capwap_nss_shim.ko` is the same shape with freeWTP's `smartcapwap_wtp` names.

### E. The original blocker is removable
- `freewtp/kernel-patches/mac80211_packet_tunnel-linux-4.8.patch` touches only `net/mac80211/{rx,tx,iface}.c` + `include/net/mac80211.h` — none of which is in the IPQ6018 radio datapath (`qsdk/qca/src/qca-wifi/`). Hence delete, don't port.

---

## 8. Build / integration

1. Enable in QSDK: `kmod-qca-nss-clients` (provides `qca-nss-capwapmgr`, `qca-nss-dtlsmgr`), `qca-wifi`, `qca-hostap`.
2. Package freeWTP for OpenWrt (it already has `freewtp/openwrt/` recipes) **without** `kmod/` and **without** the mac80211 patches feed.
3. Add `capwap_nss_shim.ko` as a small kmod package depending on `kmod-qca-nss-clients`; it `#include`s `nss_capwapmgr.h` and registers the `smartcapwap_wtp` genl family.
4. Config plumb: AC-dictated SSID/security → qca-hostapd config; AC peer/MTU/DTLS → freeWTP → shim → NSS tunnel.

---

## 9. Risks & mitigations

| Risk | Mitigation / evidence |
|---|---|
| Dual MLME conflict if freeWTP drives radio | Don't — qca-hostapd owns radio (Local-MAC). §6, Proof C |
| Downlink next-hop wiring (decap→VAP) | Start with VAP+capwap0 bridge; explicit next-hop API exists (§5) |
| Split-MAC (raw 802.11) desired by AC | Supported by `NSS_CAPWAP_PKT_TYPE_802_11` + qca-wifi raw VAP, but defer — higher cost |
| freeWTP `kmod.c` assumes synchronous genl semantics | Shim mirrors `nss_nlcapwap.c` sync request/response model (Proof D) |
| API drift across QSDK versions | Pin to this tree's `exports/` headers; all cited symbols are `EXPORT_SYMBOL` in 12.2 |

---

## 10. Milestones

1. **M0 — Tunnel bring-up (no WiFi):** shim + `nss_capwapmgr_ipv4_tunnel_create`/`enable`; verify CAPWAP data keepalive to a test AC via `nss_capwapmgr_tunnel_stats`.
2. **M1 — Local-MAC data:** create a qca-hostapd BSS; bridge VAP↔`capwap0`; confirm a STA's traffic reaches the AC through NSS (offloaded).
3. **M2 — freeWTP control integration:** freeWTP FSM drives M0+M1 from real AC Discovery/Join/Configure; map `kmod.c` calls onto the shim.
4. **M3 — DTLS offload:** move control/data DTLS to `nss_capwapmgr_configure_dtls` (v2.0, IPQ60xx).
5. **M4 — Hardening:** flow rules, MTU/fragmentation, stats, recovery.

---

*Generated from source inspection of this QSDK 12.2 tree and the cloned freeWTP. All citations verified against the files present in the repo.*