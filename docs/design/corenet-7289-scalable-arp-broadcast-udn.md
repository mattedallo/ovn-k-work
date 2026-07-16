# Scalable ARP and Broadcast Handling for UDN

**Epic:** CORENET-7289
**Status:** Draft
**Related:** OCPBUGS-85627 (root cause), CORENET-6996 (GARP storm), CORENET-7290 (long-term topology change)

## Problem Statement

ARP and broadcast traffic on br-ex is flooded to all UDN patch ports. Each primary UDN adds a patch port, and each patch port traversal costs around 100 OVS resubmits (the exact count varies by OVN version and enabled features). This flooding is unnecessary (no UDN GR legitimately needs to see another UDN's ARP or inbound ARP replies) and has two consequences that worsen with UDN count:

1. **Packet drops at moderate UDN counts:** Each time a flooded ARP packet enters a UDN patch port, it traverses the full OVN logical pipeline for that UDN (~100 resubmits). With N UDN patch ports, a single flooded packet costs ~100 x N resubmits. OVS enforces a hard-coded 4,096 per-packet resubmit limit, which is reached at roughly 35-50 UDNs. Above this threshold OVS drops the packet, breaking ARP resolution, preventing SNAT flow programming, and causing external connectivity failures.

2. **CPU waste:** Even when N is small enough to stay under 4,096, every flooded ARP packet still traverses all N pipelines unnecessarily. This consumes CPU cycles proportional to UDN count, which could cause performance degradation at high broadcast rates.

The flooding occurs through two paths:

- **Unicast ARP replies** destined to the node's physical NIC MAC (shared by all UDN Gateway Routers). An explicit `output:patch1,...,patchN,NORMAL` rule at priority 10 floods to all patches.
- **Broadcast ARP / multicast NDP** -- the `NORMAL` action floods to all ports on br-ex including all N patches.

## Scope

This is a **short-term fix** intended to be delivered quickly with minimal complexity. Solutions that require OVN core changes (schema/northd/ovn-controller modifications), topology rework, or deeper architectural changes are deferred to CORENET-7290 as a separate long-term effort.

## Goals

- Scale to 500 primary UDNs per node without hitting the 4,096 resubmit limit.
- Eliminate O(N) ARP/broadcast fan-out across UDN pipelines on br-ex.
- Handle both IPv4 ARP and IPv6 NDP.
- Deliver with low side-effect risk.

## Non-Goals

- Changing the shared MAC architecture (CORENET-7290).
- OVN core changes (northd, ovn-controller, SB-DB/NB-DB schema).
- Addressing non-ARP/NDP broadcast traffic.
- DPU mode support (different architecture where the representor port replaces LOCAL).
- Reducing OVN pipeline depth.

## Background: Current Architecture

### Per-UDN External Topology

Each primary UDN creates a complete, isolated set of OVN logical objects per node. On the external side, every UDN gets its own Gateway Router (GR) and external logical switch (ext\_LS), connected to br-ex via a localnet/patch port:

```
                          Physical Network
                                |
                          [ br-ex / ens5 ]
                        MAC: 06:38:d6:0b:4b:a7    (node's physical NIC MAC)
                                |
                  OVS bridge with per-network patch ports
                 /              |              \              \
   [ext_<node>]      [ext_net1_<node>]    [ext_net2_<node>]   ...
   (localnet)         (localnet)           (localnet)
        |                  |                    |
  [GR_<node>]       [GR_net1_<node>]    [GR_net2_<node>]      ...
  default network   UDN "net1"          UDN "net2"
```

All ext\_LS localnet ports map to the same physical network (`"physnet"` → br-ex).

### The Shared MAC

All GR external ports (`rtoe-GR_*`) use the **same MAC address**, the node's physical NIC MAC.

Giving each GR a unique MAC is a significantly more complex change that is being evaluated as part of the long-term topology redesign (CORENET-7290).

### How ARP and NDP Are Currently Forwarded

The current flow table handles ARP and NDP as follows:

#### Outbound ARP (GR resolves upstream gateway)

When a UDN GR needs to reach an external IP, it must first resolve the upstream gateway's MAC via ARP. The GR sends a broadcast ARP request using its external port identity: the shared MAC and node IP.

1. ARP request exits the UDN's ext\_LS localnet port onto br-ex.
1. The priority-10 egress flow matches (`in_port=<patch>, dl_src=bridgeMAC → output:NORMAL`).
1. `NORMAL` action performs standard L2 broadcast flooding, the ARP goes to `ofPortPhys` (correct) but **also** to all other N-1 UDN patch ports (unnecessary).
1. Each receiving UDN's ext\_LS processes the broadcast through its full OVN pipeline (~100 resubmits per pipeline, varies by OVN version).

#### Inbound Unicast ARP Reply (Gateway Responds)

The upstream gateway sends a unicast ARP reply addressed to the shared MAC.

1. Reply arrives on br-ex via `ofPortPhys`.
1. It matches the priority-10 unicast flood rule: `dl_dst=<bridgeMAC> → output:patch1,patch2,...,patchN,NORMAL`
1. This rule explicitly outputs to **ALL** N patch ports sequentially.
1. Each patch port delivers the ARP into its ext\_LS, which forwards to its GR. OVS processes all N pipelines sequentially.
1. At high UDN counts: total resubmits exceed 4,096 → OVS drops the packet.

This unicast flood rule exists because all GRs share the same MAC, OVS MAC learning cannot determine which single patch port should receive the frame.

#### Inbound Broadcast ARP (External Host Asks "Who Has NodeIP?")

1. Broadcast ARP request arrives on br-ex via `ofPortPhys`.
2. No specific flow matches (broadcast + ARP + no `in_port` restriction at priorities above 10).
3. Falls to priority-0 `NORMAL` catch-all, which floods to all ports including all N patch ports.
4. Each UDN GR sees the request, recognizes the node IP on its external port, and generates an ARP reply, producing N duplicate replies on the physical network.

#### NDP (IPv6 Neighbor Discovery)

NDP follows the same three-way pattern as ARP (outbound resolution, inbound reply, inbound external query) with these differences:

- **Multicast instead of broadcast:** NDP NS uses solicited-node multicast (`dl_dst=33:33:ff:xx:xx:xx`) rather than L2 broadcast, but `NORMAL` floods it to all ports identically.
- **Conntrack interception:** Inbound unicast NDP NAs (`icmpv6_type=136`) are IPv6 and match the priority-50 conntrack flow (`ipv6, dl_dst=bridgeMAC → ct(table=1)`) *before* the priority-10 unicast flood rule. After conntrack, a priority-14 `FLOOD` action in table 1 sends the NA to all N patches. This is **worse** than the ARP path because it adds conntrack overhead on top of the per-pipeline cost.
- **Duplicate NAs:** Inbound solicited-node multicast NS from external hosts falls to the priority-0 `NORMAL` catch-all, producing N duplicate NAs on the physical network (same as the duplicate ARP replies problem).

#### Current br-ex Flow Table (Priority 9-50, Table 0)

| Pri | Match | Action | Purpose |
|-----|-------|--------|---------|
| 50 | `ip/ipv6, dl_dst=<bridgeMAC>` | `ct(zone=...,nat,table=1)` | IP/IPv6 return traffic → conntrack (NDP NAs hit this) |
| 14 | *(table 1)* `icmp6, icmpv6_type=136` | `FLOOD` | *(table 1)* NDP NA flood after conntrack |
| 10 | `dl_dst=<bridgeMAC>` | `output:patch1,...,patchN,NORMAL` | Unicast flood to all patches (the problem for ARP replies) |
| 10 | `in_port=<patch>, dl_src=<bridgeMAC>` | `output:NORMAL` | OVN non-IP egress (broadcasts and multicast flood via NORMAL) |
| 9 | `in_port=<patch>` | `drop` | Drop wrong-MAC traffic from OVN |
| 0 | *(catch-all)* | `NORMAL` | Standard L2 forwarding (floods broadcasts and multicast) |

## Design Overview

Three mechanisms work together to eliminate ARP/NDP fan-out. The IPv4 traffic scenarios are shown separately below; IPv6 NDP is handled via [MAC_Binding propagation](#ipv6-ndp-resolution-via-mac_binding-propagation).

### Scenario 1: Outbound ARP from UDN GR

```
  UDN patch port
       │
  broadcast ARP request (arp_op=1, dl_src=bridgeMAC)
       │
       ▼
  ┌─ pri 40: ARP Proxy (dynamic) ────────────────┐
  │  arp_op=1, arp_tpa=<known IP>                │
  │                                              │
  │  Known neighbor?                             │
  │    YES → construct ARP reply → IN_PORT ──────┼──▶ back to requesting GR
  │          O(1). No wire traffic.              │    (GR updates MAC_Binding)
  │                                              │
  │    NO  → no match, fall through              │
  └──────────────────┬───────────────────────────┘
                     │
                     ▼
  ┌─ pri 15: Broadcast Isolation ───────────────┐
  │  in_port=<UDN_patch>, broadcast ARP         │
  │  → output:ofPortPhys ───────────────────────┼──▶ to physical network
  └─────────────────────────────────────────────┘    (real ARP on wire)

  ✗ NOT flooded to other UDN patches
  ✗ NOT sent to LOCAL
  ✗ NOT broadcast via NORMAL action
```

### Scenario 2: Inbound Unicast ARP Reply

```
  Physical network (ofPortPhys)
       │
  unicast ARP reply (arp_op=2, dl_dst=bridgeMAC)
       │
       ▼
  ┌─ pri 45: Uplink ARP Steering ───────────────┐
  │  in_port=ofPortPhys, arp                    │
  │  → output:default_patch, LOCAL              │
  └──────────┬────────────────────┬─────────────┘
             │                    │
             ▼                    ▼
        default_patch           LOCAL
        (default GR creates     (kernel maintains its own
         MAC_Binding in SB-DB)   table for host networking)
              │
         SB-DB event → proxy flow updated (pri 40)
         → future UDN ARP answered locally

  ✗ NOT flooded via output:patch1,...,patchN (intercepted above pri 10)
  ✗ UDN patches never see inbound ARP replies
```

### Scenario 3: Inbound Broadcast ARP ("who has nodeIP?")

```
  Physical network (ofPortPhys)
       │
  broadcast ARP request (dl_dst=ff:ff:ff:ff:ff:ff)
       │
       ▼
  ┌─ pri 45: Broadcast Isolation ───────────────┐
  │  in_port=ofPortPhys, broadcast ARP          │
  │  → output:default_patch, LOCAL              │
  └──────────┬────────────────────┬─────────────┘
             │                    │
             ▼                    ▼
        default_patch           LOCAL
        (default GR learns       (kernel responds
         MAC bindings)            for node IP,
                                  EgressIPs, etc.)

  ✗ NOT flooded to UDN patches (pri 45 > pri 0 NORMAL)
  ✗ No N duplicate ARP replies on the physical network
```

On first resolution for an unknown IP, the UDN GR's ARP goes to the wire (priority 15), the reply is steered to LOCAL + default\_patch (priority 45), the default GR's pipeline learns the neighbor and creates a `MAC_Binding` in SB-DB, the watcher programs the proxy flow (priority 40), and subsequent ARP from any UDN GR for that IP is answered locally. See [Bootstrap: First ARP Resolution](#bootstrap-first-arp-resolution) for the full sequence.

### Summary

1. **Traffic Isolation and Steering** (static flows, pri 15/45/52): Prevents ARP/NDP from flooding between patch ports. UDN patch broadcasts go to the physical uplink only. Uplink ARP/NDP goes to LOCAL + default patch only. NDP NAs are steered at priority 52 above conntrack.

2. **ARP Proxy** (dynamic flows, pri 40): An SB-DB MAC_Binding watcher programs ARP responder flows on br-ex that answer requests locally for known neighbors.

3. **IPv6 NDP Resolution via MAC_Binding Propagation**: The default GR's resolved MAC bindings are propagated to UDN GRs via direct SB-DB writes, leveraging OVN's built-in traffic-based lifecycle for automatic keepalive and cleanup.

## Detailed Design

### Traffic Isolation and Steering

**Location:** Static flows generated in `commonFlows()` in [bridgeflows.go](../../go-controller/pkg/node/bridgeconfig/bridgeflows.go).

**Guard:** only installed when the ARP proxy feature is fully active. This requires all of the following: (1) UDN support is active (`IsNetworkSegmentationSupportEnabled()`), (2) `enable-arp-proxy=true` in config, (3) not in DPU mode. Without UDNs there is at most one patch port and no fan-out problem. When any guard fails, these flows are omitted and the existing priority-10/0 rules handle all traffic unchanged.

The following static flows are installed (see [Complete Flow Priority Table](#complete-flow-priority-table-br-ex-table-0) for the full picture with match/action details): priority-52 steers inbound NDP NAs above conntrack; priority-45 steers all uplink ARP/NDP NS to `default_patch` + LOCAL and routes all default-patch/LOCAL ARP and NDP NS to the wire; priority-15 sends UDN patch broadcasts to the wire only. One priority-15 flow pair (ARP + NDP NS) is generated per UDN patch port; priority-45/52 flows are generated once.

**Why the uplink flow matches ALL ARP (not just broadcast):** External devices can send unicast ARP requests (`dl_dst=bridgeMAC, arp_op=1`) for neighbor reachability confirmation (NUD probes). If the uplink flow only matched broadcast, these unicast requests would fall to the priority-40 ARP proxy, which would incorrectly answer them acting as an ARP proxy for addresses it doesn't own. Matching all ARP from the uplink at priority 45 ensures the proxy only sees ARP from UDN patch ports.

This also means the priority-45 uplink flow catches both ARP requests AND replies, making the separate priority-15 unicast ARP reply steering flow (`arp_op=2, dl_dst=bridgeMAC`) unnecessary for traffic from the uplink. The priority-15 flows are retained for UDN broadcast isolation only.

**Why the LOCAL flows match ALL ARP (not just broadcast):** The kernel's own ARP (solicited requests and NUD probes) must reach the physical network for real neighbor verification, not be short-circuited by the proxy. If these flows only matched broadcast ARP, the kernel's unicast NUD probes would fall to the priority-40 ARP proxy and be answered locally, thus the kernel would never detect an unreachable neighbor because the proxy always responds with whatever MAC SB-DB holds. Matching all ARP from LOCAL at priority 45 ensures the kernel can properly validate its own neighbor table via real wire-level exchanges.

**NDP NS match criteria:** NDP NS uses solicited-node multicast (`dl_dst=33:33:ff:xx:xx:xx`), not L2 broadcast. The flows match `icmp6, icmpv6_type=135` without a `dl_dst` restriction to catch both solicited-node multicast and unicast NS (NUD probes).

#### Design Rationale

**Why priority 45 for non-UDN sources:** These flows must be above the ARP proxy (priority 40) so that traffic from the uplink, LOCAL, and the default patch port is always forwarded to its real destination and never intercepted by the proxy. The kernel on breth0 must see real ARP traffic from the uplink to maintain its own neighbor table. The default GR must receive broadcast ARP to respond for service ExternalIPs and other VIPs.

**Why priority 15 for UDN patches:** Below the ARP proxy (priority 40), so UDN GR ARP requests are first checked against known neighbors (proxy answers locally if known). Only if the proxy has no match does the request fall to priority 15 and get forwarded to the physical uplink for real resolution. Above the priority-10 `NORMAL` catch-all and the priority-10 unicast flood rule.

**Why UDN ARP/NS do NOT go to LOCAL:** A UDN GR's broadcast ARP uses `arp_spa=nodeIP, dl_src=bridgeMAC` which is identical to the host's own identity. If sent to LOCAL, the kernel would see an ARP from "itself," which serves no purpose. Similarly, a UDN GR's NDP NS uses solicited-node multicast (`dl_dst=33:33:ff:xx:xx:xx`) with `dl_src=bridgeMAC` which is also the host's own identity. The kernel maintains its neighbor table from real uplink traffic (priority-45 `in_port=ofPortPhys` flow).

**Why the default patch port is excluded from restriction:** The default network's GR is the only non-UDN patch port and there is exactly one per node, so it does not contribute to the O(N) fan-out that this design eliminates. Placing it at priority 45 alongside the uplink and LOCAL sources preserves its pre-existing behaviour (all ARP goes directly to the wire, bypassing the proxy) and keeps the flow table conceptually clean: all non-UDN sources at priority 45, all UDN sources at priority 15. Matching all ARP (not just broadcast) ensures that OVN stale probes (unicast ARP sent by ovn-controller for MAC\_Binding liveness on OVN 25.03+) reach the physical network for real verification rather than being short-circuited by the proxy. After CORENET-6996 removes `nat-addresses=router` from UDN ext\_LS ports, the default GR is also the only one that sends GARPs for NAT addresses, and keeping it above the proxy avoids any interaction with proxy matching.

**Why NDP NAs are steered at priority 52 (above conntrack):** The existing priority-50 flow `ipv6, dl_dst=bridgeMAC → ct(table=1)` intercepts all unicast IPv6 before any lower-priority flow. A priority-15 NDP NA flow would never match unicast NAs. The fix is to steer NDP NAs at priority 52, above conntrack.

**Why the NDP NA flow omits `dl_dst`:** NDP NAs arrive with different Ethernet destinations depending on context: solicited NAs (replies to an NS) use unicast `dl_dst=bridgeMAC`, while unsolicited NAs (e.g., a router announcing a MAC change) use multicast `dl_dst=33:33:00:00:00:01` (ff02::1, all-nodes). If the flow restricted to `dl_dst=bridgeMAC`, multicast unsolicited NAs would bypass priority 52, miss the priority-50 conntrack flow (which also requires `dl_dst=bridgeMAC`), and fall to priority-0 `NORMAL`, flooding all N UDN patches. Omitting `dl_dst` catches both variants, consistent with how the priority-45 ARP and NS flows already omit `dl_dst` for the same reason.

### ARP Proxy on br-ex

**Location:** New file (e.g. `go-controller/pkg/node/arp_proxy_linux.go`).

**Guard:** Same as [Traffic Isolation and Steering](#traffic-isolation-and-steering).

**Source of truth:** The default GR's `MAC_Binding` entries in OVN SB-DB. When an ARP reply is steered to `default_patch` by the priority-45 flow, the default GR learns the neighbor and creates a `MAC_Binding` entry in SB-DB. The `macBindingWatcher` observes these entries and programs corresponding ARP responder flows on br-ex.

The default GR learns from any ARP packet that reaches its external port, regardless of whether it originated the request. This means the proxy also preemptively learns external hosts' MACs from inbound broadcast ARP requests (Scenario 3), providing immediate proxy responses when a UDN GR later needs to reach that host.

##### System Actors and Responsibilities

The ARP proxy involves three actors:

- **macBindingWatcher** [NEW] — Watches the default GR's `MAC_Binding` entries in SB-DB with server-side filtering (only the local node's default GR port). On ADD or UPDATE (MAC change): generates the proxy flow and writes to the flow cache. On DELETE: removes the corresponding flow.
- **openflowManager** [EXISTING] — Owns the aggregated flow cache for ALL producers (DEFAULT, NORMAL, services, EgressIP, ...), atomic OVS sync, and the 15s periodic safety-net re-sync. Does not generate proxy flows, does not notify producers of sync success or failure. The macBindingWatcher is just another producer of flows.
- **OVN SB-DB (MAC\_Binding table)** — Holds the default GR's resolved neighbors. Entries are created by ovn-controller when the GR receives ARP traffic, aged by northd (`mac_binding_age_threshold=300`), and refreshed by `mac_cache_use` or stale probes.


##### IPv4 ARP Responder Flows (priority 40)

For each known IPv4 neighbor `(IP, MAC)` from the default GR's MAC_Binding table, the macBindingWatcher defines the following flow:

```
cookie=<ARPProxyCookie>, priority=40, table=0, arp, arp_op=1, arp_tpa=<IP>,
  actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],
          set_field:<MAC>->eth_src,
          set_field:2->arp_op,
          move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],
          move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],
          set_field:<MAC>->arp_sha,
          set_field:<IP>->arp_spa,
          IN_PORT
```

This constructs a valid ARP reply in-place and sends it back to the requesting port via `IN_PORT`.

**Why `IN_PORT`:** The request can only come from a UDN patch port (uplink, LOCAL, and default patch traffic is caught at priority 45). `IN_PORT` sends the reply back to the exact UDN GR that asked, which updates its `MAC_Binding` table in OVN SB.

### IPv6 NDP Resolution via MAC_Binding Propagation

Unlike IPv4 where the ARP proxy constructs a reply in OpenFlow (and the GR creates its own `MAC_Binding`), a pure-OpenFlow NDP responder cannot be built on br-ex due to OVS kernel datapath limitations (see [Pure-OpenFlow NDP Responder on br-ex](#pure-openflow-ndp-responder-on-br-ex) in Alternatives Considered). Instead, the macBindingWatcher writes `MAC_Binding` entries directly to SB-DB for each UDN GR's external port when the default GR resolves an IPv6 neighbor. These entries are dynamic (subject to `mac_binding_age_threshold=300`) and benefit from OVN's built-in lifecycle management.

#### Overview

1. The default GR resolves an IPv6 neighbor via NDP. This creates a dynamic `MAC_Binding` entry in SB-DB.
2. The macBindingWatcher observes the default GR's `MAC_Binding` entries in SB-DB.
3. When an IPv6 binding appears or its MAC changes, the watcher writes a `MAC_Binding` entry directly to SB-DB for each UDN GR's external port (`rtoe-GR_*`) for that `(IP, MAC)` pair with a fresh timestamp.
4. Bidirectional UDN traffic (e.g. TCP) keeps entries alive via `MAC_CACHE_USE` (return traffic refreshes timestamp). Entries never expire while bidirectional traffic flows.
5. Idle entries expire after 300s (without controller involvement).
6. When UDN traffic resumes after idle: NS sent on wire → default GR re-learns → controller re-creates entries. OVN buffers the first packet.

**Scale:** This produces `N x M` MAC_Binding entries per node, where N is the number of UDNs and M is the number of IPv6 neighbors the default GR has resolved. M is driven by the GR's **connected route** for its external subnet. The GR's external port (`rtoe-GR_*`) is assigned the node's IP with a prefix length (e.g., `fd2e:.../64`), which creates an implicit connected route for the entire subnet. Since all cluster nodes sit on the same external subnet, the best-case M = default gateway + number of nodes. At 500 UDNs and 500 nodes, that is ~250K MAC_Binding entries per node. Every resolved binding is propagated to **all** UDN GRs regardless of which UDN triggered the resolution (the returning NA is steered to `default_patch` with no correlation to the originating UDN).

**Bootstrap timing:** If a UDN GR needs to reach an IPv6 neighbor before the MAC_Binding has been propagated (e.g., UDN created before the default GR has resolved, or after an entry has expired), the UDN GR sends NDP NS which goes to the wire (priority-15). The gateway's NA is steered to `default_patch` + LOCAL (priority-52). The UDN GR does NOT receive this NA directly, only the default GR does (via `default_patch`). The default GR learns the binding, the controller sees the ADD event and propagates to UDN GRs. OVN buffers the original packet for up to 10 seconds; this should give enough time for the MAC bindings to be created and buffered packets to be released without loss.

#### Controller Architecture

**Location:** Unified macBindingWatcher in the ovnkube-node process. The same watcher that programs IPv4 ARP proxy flows also handles IPv6 MAC_Binding propagation, taking different actions depending on the `IP` field:
- **IPv4 bindings:** Generate ARP proxy flow strings → write to openflowManager flow cache
- **IPv6 bindings:** Write `MAC_Binding` entries for UDN GRs through the SB-DB client.

**SB-DB client:** A SB-DB client on the node process connects to the local SB-DB via Unix socket, monitoring only `MAC_Binding` with server-side conditional filtering on `logical_port == "rtoe-GR_<node>"`. This ensures only the local node's default GR entries are cached, avoiding the cluster-wide MAC_Binding table (which could be millions of rows at scale).

#### Event Handling

- **ADD** (new IP resolved): Write `MAC_Binding` for all UDN GRs with same `(IP, MAC)` and fresh timestamp in a single batched transaction. ovn-controller installs flows incrementally (no northd involvement).
- **UPDATE** (MAC changed): Update all UDN GR MAC_Bindings with the new MAC.
- **UPDATE** (timestamp refreshed, same MAC): Write a fresh timestamp to all UDN GR MAC_Bindings for that IP. This keeps UDN GR gateway bindings alive when `mac_cache_use` cannot refresh them directly. Batched; at most one write per UDN GR per refresh cycle.
- **DELETE** (default GR binding aged out): No action needed. UDN GR entries have independent timestamps and are managed by OVN's own lifecycle:
  - If UDN traffic is bidirectional → `MAC_CACHE_USE` keeps the UDN entry alive independently.
  - If UDN traffic is idle → the UDN entry ages out on its own. On next traffic, the default GR re-resolves → ADD event → re-propagated.

**Potential improvement:** If timestamp-driven write amplification becomes a concern at extreme scale, the controller could switch to periodic reconciliation: sweep every half binding expiration time, refresh UDN entries approaching expiry in one batch. 


#### Lifecycle Hooks

| Trigger | Action |
|---------|--------|
| New UDN created | Query current default GR MAC_Bindings → batch-create entries for new UDN GR |
| UDN deleted | northd handles cleanup (deletes MAC_Bindings for removed datapaths via strong `datapath` reference) |
| Node process restart | libovsdb reconnects with full dump → watcher re-creates any missing UDN GR entries |

**UDN deletion:** Each `MAC_Binding` entry we create holds a strong UUID reference (`datapath` column) to the UDN GR's `Datapath_Binding`. When the UDN is deleted and northd removes the `Datapath_Binding`, OVSDB's referential integrity automatically garbage-collects all `MAC_Binding` rows that reference it. No explicit cleanup by the controller is needed.

#### Corner Cases

- **Thundering herd:** If northd batch-deletes expired entries for all 500 UDN GRs (e.g., all timestamps aligned), each UDN GR sends NS on next traffic. The neighbor receives up to 500 NS. Controller sees one ADD on the default GR → re-creates 500 entries in one batch transaction. Mitigated by northd's `mac_binding_removal_limit` option which caps deletions per sweep.
- **MAC change without unsolicited NA:** Default GR's stale probe detects MAC changes within ~56-112s for active-traffic entries by probing the neighbor and learning the new MAC from the response. For idle entries: the entry expires → next traffic re-resolves with the correct MAC.
- **Stale probe on UDN GR port:** ovn-controller detects active outbound on the UDN GR (OFTABLE_MAC_BINDING flow active), sends NS from the UDN GR's port. The NA response goes to `default_patch` (priority-52), so the UDN entry's timestamp is NOT refreshed directly. No harm — the controller mirrors the default GR's timestamp refresh to UDN GR entries (via the UPDATE/timestamp event), keeping them alive as long as the default GR's binding is active.
- **macBindingWatcher crash (node process restart):** OVN retains installed MAC_Binding flows until entries age out. On restart, libovsdb reconnects and delivers the full current state as ADD events. The watcher reconciles: re-creates any missing UDN entries and reprograms proxy flows from current SB-DB state. Entries that aged out during downtime are re-created on next UDN traffic via the bootstrap path.


## Complete Flow Priority Table (br-ex Table 0)

The following shows how new flows (marked with **NEW**) fit within the existing priority structure. Only the relevant priority range (9-52) is shown; flows at priorities 99-700 are unchanged and omitted.

| Pri | Match | Action | Status |
|-----|-------|--------|--------|
| **52** | `in_port=ofPortPhys, [matchVLAN,] icmp6, icmpv6_type=136` | `output:<default_patch>,[strip_vlan,]output:LOCAL` | **NEW** -- ALL uplink NDP NA to default GR + kernel (above conntrack) |
| 50 | `ip/ipv6, dl_dst=<bridgeMAC>` | `ct(zone=...,nat,table=1)` | Existing -- IP/IPv6 return traffic to conntrack |
| **45** | `in_port=ofPortPhys, [matchVLAN,] arp` | `output:<default_patch>,[strip_vlan,]output:LOCAL` | **NEW** -- ALL uplink ARP to default GR + kernel |
| **45** | `in_port=ofPortPhys, [matchVLAN,] icmp6, icmpv6_type=135` | `output:<default_patch>,[strip_vlan,]output:LOCAL` | **NEW** -- ALL uplink NDP NS to default GR + kernel |
| **45** | `in_port=<default_patch>, arp` | `output:ofPortPhys` | **NEW** -- ALL default GR ARP to wire |
| **45** | `in_port=<default_patch>, icmp6, icmpv6_type=135` | `output:ofPortPhys` | **NEW** -- Default GR NDP NS to wire |
| **45** | `in_port=LOCAL, arp` | `[modVLANID,]output:ofPortPhys` | **NEW** -- ALL host ARP to wire |
| **45** | `in_port=LOCAL, icmp6, icmpv6_type=135` | `[modVLANID,]output:ofPortPhys` | **NEW** -- Host NDP NS to wire |
| **40** | `arp, arp_op=1, arp_tpa=<knownIP>` | ARP reply via `IN_PORT` | **NEW** -- ARP proxy (dynamic, per neighbor) |
| **15** | `in_port=<UDN_patch>, dl_dst=ff:.., arp` | `output:ofPortPhys` | **NEW** -- UDN broadcast ARP to wire only |
| **15** | `in_port=<UDN_patch>, icmp6, icmpv6_type=135` | `output:ofPortPhys` | **NEW** -- UDN NDP NS to wire only |
| 10 | `dl_dst=<bridgeMAC>` | `output:patch1,...,patchN,NORMAL` | Existing -- Unicast flood (now harmless fallback) |
| 10 | `in_port=<patch>, dl_src=<bridgeMAC>` | `output:NORMAL` | Existing -- OVN non-IP egress |
| 9 | `in_port=<patch>` | `drop` | Existing -- Drop bad MAC from OVN |
| 0 | *(catch-all)* | `NORMAL` | Existing -- Default L2 forwarding |

The priority-10 unicast flood rule becomes a harmless fallback. ARP from the uplink is intercepted at priority 45. NDP NAs (both unicast and multicast) are intercepted at priority 52. Non-ARP unicast to `bridgeMAC` is already handled by priority-50+ conntrack flows.

## Edge Cases, Interactions, and Failure Modes

### Bootstrap: First ARP Resolution

When a UDN GR first needs to resolve an unknown IP (e.g., the upstream gateway):
1. GR sends an ARP request (broadcast).
2. The ARP proxy has no entry at priority 40 -- no match.
3. Priority-15 flow sends the request to `ofPortPhys` (physical network).
4. The gateway responds with a unicast ARP reply.
5. Priority-45 flow (`in_port=ofPortPhys, arp`) steers the reply to LOCAL + default\_patch.
6. The default GR learns the neighbor and creates a `MAC_Binding` in SB-DB. The macBindingWatcher observes the ADD event and programs the proxy flow.
7. The requesting UDN GR did NOT receive this first reply.

**OVN buffers the original IP packet** that triggered the ARP request (for up to 10 seconds, with limits of 1000 unique destinations and 4 packets per destination). When a subsequent packet for the same destination triggers a new ARP request, the proxy answers immediately through the new installed flow. The GR creates a dynamic `MAC_Binding` in SB, and ovn-controller reinjects the buffered packets. If the proxy flow is not yet installed when the retry arrives, the ARP simply takes the bootstrap path again.


### MAC_Binding Lifecycle and Proxy Flow Availability

The ARP proxy flow for a given IP exists on br-ex as long as the default GR has a corresponding `MAC_Binding` entry in SB-DB. The proxy flow's lifecycle is therefore directly tied to the default GR's MAC_Binding lifecycle, which depends on the type of destination:

**Default gateway** (for any off-subnet traffic): The default GR's table-66 flow (`OFTABLE_MAC_BINDING`) is active because default-network pods route off-subnet traffic through it. On OVN 25.03+ ([commit `58ce60d`](https://github.com/ovn-org/ovn/commit/58ce60d2f1d932b842512763c2b8fc0943e1f8e3)), ovn-controller sends background ARP probes (every `3/16 × threshold ≈ 56s`) that keep the binding alive indefinitely. On OVN 24.09 and earlier: the binding expires every 300s and is re-created via the bootstrap path on next traffic (the proxy flow is briefly absent, OVN buffers the packet).

**Same-subnet destinations with default-network traffic** (e.g., other cluster nodes): If the default GR receives return data traffic with `nw_src=<destination_IP>`, the [`mac_cache_use`](https://github.com/ovn-org/ovn/commit/33bb66c6c8e6e119bf9006dbe868457eecf82c9e) flow refreshes the binding timestamp. The binding never expires while bidirectional traffic flows through the default GR.

**UDN-only same-subnet destinations** (external hosts that only UDN pods talk to): The default GR has no independent outbound traffic to these IPs, so neither `mac_cache_use` nor stale probes refresh the binding. The binding expires predictably at 300s. When a UDN GR next needs to reach this IP, the ARP takes the [bootstrap path](#bootstrap-first-arp-resolution): ARP → wire → reply → default GR re-learns → proxy flow re-installed. OVN buffers the packet (10s, 4/dest).

**GARP detection:** When an external device sends a GARP (announcing a MAC change), the priority-45 flow steers it to `default_patch`. The default GR updates its MAC\_Binding with the new MAC. The macBindingWatcher sees the UPDATE event and reprograms the proxy flow.

**UDN GR stale probes answered by proxy (OVN 25.03+):** When ovn-controller detects active outbound on a UDN GR port (table 66 flow activity), it sends a unicast ARP probe from the UDN GR's external port. This probe hits the priority-40 proxy flow and is answered locally. The UDN GR therefore delegates liveness verification to the default GR indirectly — the proxy flow only exists while the default GR's binding exists, and the default GR performs real probing on the physical network. If the default GR's binding expires, the proxy flow is removed, and the UDN GR's next probe falls to priority-15 → wire → real re-resolution. The system self-heals.

**Temporary proxy flow absence:** When a default GR binding expires and is later re-created (via bootstrap), there is a brief window without a proxy flow. This does not affect active UDN data traffic — the UDN GR still has its own valid MAC\_Binding and forwards data normally using the known MAC. For same-subnet destinations, `mac_cache_use` from return traffic independently keeps the UDN GR's binding alive regardless of the proxy flow's state.


### Process and Sync Failures

- **Process failure:** In case of ovnkube-node crash, the OVS daemon retains its last-installed flow set (including proxy flows) on br-ex, so the datapath continues forwarding autonomously. On restart, the libovsdb client reconnects using `monitor_cond_since` which replays changes since the last known transaction ID. If the server does not support replay (or the transaction ID is stale), it falls back to a full initial table dump. The macBindingWatcher repopulates the flow cache from the current SB-DB state and triggers a flow sync.

- **SB-DB unavailable:** The flow cache holds correct state but no new events arrive. Existing proxy flows remain functional (OVS retains them). When SB-DB reconnects, libovsdb automatically re-syncs via `monitor_cond_since` or full dump.
- **Malformed flow string (poisoned cache):** openflowManager flattens ALL cache keys into one bundle before enforcing them; one malformed flow rejects the ENTIRE update. If macBindingWatcher produces a bad flow (e.g., empty IP), it blocks flow updates for ALL producers on br-ex. OVS retains its previous flow set (functional but increasingly stale). The 15s periodic sync retries but keeps failing (same bad data in cache). This persists until a subsequent MAC_Binding event overwrites the bad entry or ovnkube-node restarts. **Mitigation:** macBindingWatcher MUST validate every flow string before writing to the cache and reject entries with empty/zero IP or MAC.
- **Sync failure (OVS unavailable):** The flow cache holds correct state but `replace-flows` fails (OVS crashed, vswitchd busy). OVS retains previous flow set. The 15s periodic sync retries automatically.
- **Event buffer overflow:** libovsdb's event processor uses a 65K-entry buffered channel. Under extreme churn, events can be dropped silently. Mitigation: conditional monitoring (`WithConditionalTable`) reduces event volume to only the local node's default GR entries (~500 entries max). Overflow is unlikely but if it occurs, the next periodic reconciliation (via the 15s flow sync comparing against the libovsdb cache) corrects any drift.


**RA FLOOD remains:** Router Advertisements (type 134) are not intercepted by this design's table-0 flows. Unicast RAs (`dl_dst=bridgeMAC`) enter the priority-50 conntrack flow and reach table 1, where the [priority-14 FLOOD](https://github.com/ovn-kubernetes/ovn-kubernetes/blob/1dec420c4abfd1fd51185184bca96fd2e7607615/go-controller/pkg/node/bridgeconfig/bridgeflows.go#L1163-L1169) sends them to all ports. Multicast RAs (`dl_dst=33:33:00:00:00:01/02`) bypass conntrack (priority-50 requires `dl_dst=bridgeMAC`) and fall to priority-0 `NORMAL`, which floods identically. Replacing the table-1 `FLOOD` with `output:LOCAL,<default_patch>` for type 134, and adding a table-0 steering flow for multicast RAs, remains a candidate for a follow-up change.

## Feature Gate

The ARP proxy and broadcast isolation flows are controlled by a feature gate `enable-arp-proxy` that allows operators to disable the functionality entirely and revert to the legacy behavior (broadcast flood to all UDN patch ports).

**Runtime toggle:** Changing the flag requires an ovnkube-node restart. On restart, the openflowManager performs a full flow sync which atomically installs or removes all proxy/isolation flows.


## Alternatives Considered

### Pure-OpenFlow NDP Responder on br-ex

IPv6 Neighbor Discovery uses ICMPv6 Neighbor Solicitation (NS, type 135) and Neighbor Advertisement (NA, type 136). Unlike ARP, NDP packets are more complex: they contain ICMPv6 headers with options (Source/Target Link-Layer Address) and require correct checksum computation.

OVS supports the following NDP-relevant OpenFlow fields:
- `nd_target` (NXM\_NX\_ND\_TARGET): The target IPv6 address in NS/NA. Writable via `set_field` on both NS and NA matches.
- `nd_sll` (NXM\_NX\_ND\_SLL): Source Link-Layer Address option (in NS). Writable via `set_field`, but only when the match includes `icmpv6_type=135`.
- `nd_tll` (NXM\_NX\_ND\_TLL): Target Link-Layer Address option (in NA). Writable via `set_field`, but only when the match includes `icmpv6_type=136`.
- `icmpv6_type`: Writable via `set_field`.

Each NDP field individually supports `set_field` when its prerequisite match is satisfied. However, OVS enforces prerequisites at **flow installation time** against the **match criteria**, not at packet execution time. Even chaining `set_field:136->icmpv6_type` before `set_field:...->nd_tll` in the actions is rejected if the match specifies `icmpv6_type=135` -- OVS does not re-evaluate prerequisites after action-side field modifications.

**The fundamental blocker** is that OVS's kernel datapath cannot write the Target Link-Layer Address (`nd_tll`) into a Neighbor Solicitation packet. An NS carries an option type 1 (Source LLA), and `nd_tll` targets option type 2 (Target LLA); OVS's `packet_set_nd()` function ([`lib/packets.c`](https://github.com/openvswitch/ovs/blob/main/lib/packets.c)) scans the packet's ND options by type byte, finds type 1 instead of 2, and **silently does nothing**. The natural workarounds each fail:

- **Prerequisite two-table trick:** `nd_tll` requires `icmpv6_type=136` in the flow match, but we match `icmpv6_type=135` (NS). A two-table workaround (match 135 in table A, `set_field:136->icmpv6_type`, `goto_table` to table B where the match says 136) bypasses the prerequisite check ([`ovs-fields(7)`](https://www.openvswitch.org/support/dist-docs/ovs-fields.7.txt)), but does not change the option type byte in the actual packet, so `packet_set_nd()` still silently skips the write.

- **`nd_options_type`:** The only field that could rewrite the option type byte (1 → 2) uses `OVS_KEY_ATTR_ND_EXTENSIONS`, which the Linux kernel OVS module [explicitly rejects](https://lkml.iu.edu/hypermail/linux/kernel/2203.1/02245.html). It is userspace-datapath only (while OVN-K uses kernel datapath).

OVN's own `nd_na` action uses the controller slow path: NS is punted to ovn-controller via `CONTROLLER`, which constructs the NA from scratch in userspace (`pinctrl.c`) and injects it via packet-out.

**Rejected because:** No path exists to construct an NA from an NS purely in the OVS kernel datapath.

### LOCAL as ARP Responder + NB-DB StaticMACBinding Propagation

Steer all ARP traffic (both requests and replies) to LOCAL only. The kernel answers ARP requests for addresses on breth0. A controller watches the kernel's neighbor table and writes `StaticMACBinding` entries into NB-DB for every UDN GR, so GRs never need to ARP themselves.

The concern is scale, with N UDNs and M neighbors, the NB-DB needs `N x M` StaticMACBinding entries per node. At 500 UDNs and 500 nodes (each a neighbor), that is 250K entries per node. Changing a single MAC (e.g., gateway failover) is an O(N) operation, one NB-DB write per UDN GR.

**Rejected in favor of br-ex ARP responder flows because:**
- **Scale:** O(N x M) NB-DB entries vs O(M) br-ex flows. The responder approach is independent of UDN count.
- **Latency:** NB-DB writes propagate through northd → SB-DB → ovn-controller vs SB-DB MAC_Binding → flow update


### NB-DB StaticMACBinding Propagation (for IPv6)

Watch the default GR's `MAC_Binding` in SB-DB. Write `StaticMACBinding` entries to NB-DB for each UDN GR. StaticMACBindings are permanent (no TTL, priority 150 in OpenFlow) and prevent UDN GRs from ever sending NDP NS. On default GR MAC_Binding deletion (aging), trigger a kernel NDP probe (`NUD_PROBE` via netlink) from breth0 to verify neighbor liveness; delete StaticMACBinding only if probe fails.

**Rejected in favor of direct SB MAC_Binding writes because:**

- **No traffic-based lifecycle:** StaticMACBinding entries are permanent. They accumulate for neighbors that are no longer being reached by any UDN. Cleanup requires either manual probing (kernel `NUD_PROBE` on DELETE events, ~100 lines of netlink infrastructure) or accepting permanent accumulation bounded by subnet size. Dynamic MAC_Binding entries self-clean via northd aging when idle — correct behavior with zero controller logic.
- **Probe infrastructure complexity:** To preserve liveness guarantees, StaticMACBinding requires a probe-on-DELETE mechanism (kernel NDP NS via netlink, NUD state monitoring, retry handling). Dynamic MAC_Binding delegates liveness to OVN's own aging and the inherent re-resolution path (buffered NS → re-creation), eliminating the probe infrastructure entirely.
- **Reconciliation complexity:** StaticMACBinding entries have no `ExternalIDs` field, making ownership tracking difficult. Startup reconciliation requires distinguishing propagated entries from dummy masquerade entries by IP range. Dynamic MAC_Binding entries are self-reconciling, the controller just re-creates from the default GR's current state.
- **northd cost:** `build_static_mac_binding_table()` is not incremental. Every NB-DB StaticMACBinding transaction triggers a full recompute of this function (iterates ALL entries). Direct SB MAC_Binding writes bypass northd entirely — ovn-controller processes them incrementally via `lflow_handle_changed_mac_bindings`.
- **Overrides dynamic bindings:** `override_dynamic_mac=true` at priority 150 prevents any mechanism from correcting a stale entry except the controller itself. If the controller misses a MAC change (crash during failover), the stale entry persists indefinitely causing a permanent black-hole that UDN GRs cannot self-heal from. Dynamic MAC_Binding at priority 100 expires naturally and is re-resolved with the correct MAC.


### Kernel Neighbor Table as ARP Proxy Source of Truth

Watch the Linux kernel's neighbor table on breth0 via netlink. Program ARP proxy flows for neighbors in usable NUD states (`NUD_REACHABLE`, `NUD_STALE`, `NUD_DELAY`, `NUD_PROBE`, `NUD_PERMANENT` with a non-empty hardware address). Require `arp_accept=2` sysctl so the kernel learns from ARP replies steered to LOCAL that it didn't originate (UDN GRs sent the request, not the kernel).

**Advantages of this approach:**
- **Lower latency:** The netlink path has fewer hops (kernel event → userspace → flow sync) compared to the SB-DB path (GR learns → SB-DB write → libovsdb notification → flow sync). Both are expected to complete well before the next data-plane packet triggers a retry, but the kernel path has fewer intermediate steps.
- **No new process-level dependency:** Does not require adding an SB-DB client to the node process. Uses the well-established `vishvananda/netlink` library already vendored.

**Rejected in favor of SB-DB MAC_Binding because:**
- **Separate infrastructure from IPv6:** The IPv6 path already uses SB-DB MAC_Binding as its source of truth. Using the kernel table for IPv4 means two separate subsystems with different failure modes, event sources, and lifecycles. Unifying on SB-DB means one watcher, one event source, shared lifecycle code.
- **Security-relevant sysctl:** `arp_accept=2` allows same-subnet IPs to inject neighbor entries into the kernel without solicitation.
- **Stale MAC served without verification:** Kernel `NUD_STALE` entries are never probed. If a neighbor silently changes its MAC, the proxy serves the stale MAC indefinitely until kernel GC fires. OVN's MAC_Binding aging (300s) forces periodic re-resolution, ensuring correctness. OVN 25.03+ stale probes actively verify active bindings every ~56s.


## Related Work

### CORENET-6996: GARP Storm and Masquerade ARP Fixes

CORENET-6996 ([PR #6346](https://github.com/ovn-kubernetes/ovn-kubernetes/pull/6346)) independently reduces the volume of unnecessary ARP/GARP traffic on br-ex by eliminating GARP generation from UDN GRs, fixing the masquerade subnet mask to prevent O(N^2) cross-UDN ARP resolution, and adding defense-in-depth flows (priority 11/12) that drop duplicate ARP replies from UDN patches. Neither design depends on the other for correctness: CORENET-6996 reduces packet volume while CORENET-7289 eliminates fan-out cost per packet.