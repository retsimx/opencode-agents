# Epic: Aegis Register-Broker Protocol (gateway)

**Sibling epic**: aegis-controller#1 — *Legacy-panel bridge*

## Goal

Evolve the Aegis gateway from a single-client mailbox WebSocket into a generic multi-consumer register broker: any number of clients connect, each receives a full snapshot on connect, and every register change is broadcast to every connected client — whether the change came from the climate controller push or from another client's write. The gateway is the sole writer to the RS-485 bus. The protocol is fully typed (DTOs for every known register) with a raw-hex escape hatch, a `read` verb, write-policy enforcement, `status` frames, and multi-unit projection. No API versioning, no legacy formats — the API evolves in place; gateway and controller change together atomically.

## Context

The current gateway is a single-session mailbox bridge shaped around one client's broadcast ABI. The endgame is to remove all vendor panel software: any client (web console, mobile app, automation hub) speaks this protocol directly, and the controller app becomes a thin bridge that keeps legacy panels working during migration. That requires a protocol that is generic (not client-shaped), multi-consumer, and complete (every register typed, every register raw-addressable).

This epic is the gateway half. The sibling controller epic consumes this protocol.

## Protocol specification (normative — inline, no external references)

### Transport & session

- Endpoint: `ws://<host>:2026/mailbox-stream`
- **Multi-consumer**: N readers + 1 writer. The gateway is the sole bus writer (one TX per ping); all writes are serialized.
- **Broker model**: the register bank is the shared truth, kept in sync with the controller via dump + `getCAN`. Every bank mutation is broadcast to every connected client as an `event`.
- **Snapshot-on-connect**: each new client receives the full typed bank, then the event stream.
- **Acks**: deferred-to-TX (success only after the frame hits the bus), batch-correct (one flush acks every msg_id in that frame), 10 s client timeout.
- Keepalive: WS ping ~30 s. Client reconnect/backoff is client-side.

### Addressing

```
{ "unit_type": "07", "unit_id": "3f2a1", "register": "03", "zone": 1 }
```

- `zone` only for zone-bearing registers (03, 04); part of the address, not the payload.
- `unit_type`/`unit_id` default to the primary unit when omitted.
- `payload` is a typed object for known registers, or a raw 14-char hex string for anything — both first-class.

### Message catalog

**Client → server**

| type | fields | purpose |
|---|---|---|
| `write` | `msg_id`, `unit_type?`, `unit_id?`, `register`, `zone?`, `payload` | typed (sparse merge) or raw (verbatim) register write |
| `read` | `msg_id`, `unit_type?`, `unit_id?`, `register`, `zone?` | read a register → `read_result` |
| `command` | `msg_id`, `action` | `resync` \| `flush_unit` |

**Server → client**

| type | fields | purpose |
|---|---|---|
| `snapshot` | `units` (multi-unit bank) | full typed state on connect / after resync |
| `event` | `unit_type`, `unit_id`, `register`, `zone?`, `payload` | register change, broadcast to all |
| `read_result` | `msg_id`, `unit_type`, `unit_id`, `register`, `zone?`, `payload` | correlated read reply |
| `ack` | `msg_id`, `status` (`success`\|`error`), `reason?` | correlated response |
| `status` | `state` (`negotiating`\|`synced`\|`resyncing`\|`link_down`), `detail?` | gateway/bus health |
| `error` | `message`, `reason?` | protocol error |

### DTOs (per register)

| Reg | DTO | Example | Write policy |
|---|---|---|---|
| 01 | `zone_config` | `{"header":32,"total_zones":4,"constant_zones":1,"constant_zone_ids":[1,0,0],"filter_clean_required":false}` | writable |
| 02 | `unit_activation` | `{"unit_type":"aegis_mk2","activation_status":"no_code","dict_fw_major":17,"dict_fw_minor":22}` | **read-only** |
| 03 | `zone_state` | `{"open":true,"damper_pct":100,"sensor_type":"rf","target_temp_c":23.0,"measured_temp_c":22.5}` | write `open`/`damper_pct`/`target_temp_c`; rest read-only |
| 04 | `zone_limits` | `{"min_damper":20,"max_damper":80,"motion_status":0,"motion_config":1,"zone_error":0,"rssi":0}` | write `min_damper`/`max_damper`/`motion_config`; rest read-only |
| 05 | `system_status` | `{"power":"on","mode":"cool","fan":"high","target_temp_c":24.0,"myzone_id":1,"fresh_air":false,"rf_sys_id":0}` | write all but `rf_sys_id` |
| 06 | `firmware` | `{"fw_major":5,"fw_minor":3,"cb_type":4,"rf_fw_major":3}` | **read-only** |
| 08 | `system_error` | `{"error_code":"AA1"}` | **read-only** |
| 09 | `activation_code` | `{"action":"set_code","unlock_code":"A1B2","activation_days":43}` | **write-only** |
| 0a | `unit_announcement` | `{}` | **read-only** |
| 12 | `sensor_pairing` | read `{"sensor_uid":"01613d","pairing":true,"sensor_rev":14}`; write `{"sensor_uid":"01613d","zone":1}` | write `sensor_uid`/`zone` |
| 26 | `rf_device_pairing` | `{"pairing_control":1,"rf_device_type":1,"zone_channel":0}` | writable |
| 27 | `rf_device_calibration` | `{"calibration_control":1,"channel":0,"up_down_position":1}` | writable |
| 16/17/1d/1e | — (opaque) | raw hex only | **read-only by default** (unverified) |

Enums: `mode` = `cool|heat|vent|auto|dry|myauto`; `fan` = `off|low|medium|high|auto|auto_aa`; `power` = `on|off`; `sensor_type` = `no_sensor|rf|wired|rf2can_booster|rf_x`; `activation_status` = `no_code|code_enabled|expired`; `unit_type` = `aegis_mk1|aegis_mk2|aegis_mk3`; `action` = `set_code|unlock`.

### Write-policy enforcement

- Whole-register read-only (02, 06, 08, 0a, 13, 16/17/1d/1e): any `write` rejected, typed or raw → `ack {status:"error", reason:"register 08 is read-only"}`.
- Field-level read-only (03/04/05/12): a typed write containing a read-only field rejected with the field named.
- Raw hex writes to a writable register bypass field checks; raw writes to a read-only register rejected.
- Write-only (09): reads rejected. Internal (07): not addressable.
- Unverified (13, 16/17/1d/1e): read-only by default.
- Range validation: mode ≤5, damper ≤100, setTemp×2 ≤80, motion ≤22, motionConfig ≤2, measDec ≤9, freshAir ≤2, rfSysId ≤16, myzone ≤10, numZones ≤10, numConstants ≤3 → error ack naming the field and value.

### Wire layouts (DTO ↔ 7-byte codec)

01 `[header][num_zones][num_constant][const1][const2][const3][filter]`
02 `[unit_type][activation][dict_fw_maj][dict_fw_min]`
03 `[zone][open|pct][sensor][set×2][meas_int][meas_dec][00]`
04 `[zone][min][max][motion][motion_cfg][error][rssi]`
05 `[power][mode][fan][set×2][myzone][fresh][rf_id]`
06 `[fw_maj][fw_min][cb_type][rf_fw_maj][00][00][00]`
08 `[5 ASCII][00][00]`
09 `[action][code_hi][code_lo][days][00][00][00]`
12 read `[uid 3B][info][rev][00][00]`; write `[uid 3B][zone][00][00][00]`
26 `[pairing][rf_type][zone_ch][00]×4`
27 `[calib][channel][pos][00]×4`

### Session behaviour

- Negotiation: `getSystemData` → `CAN2 in use` (normal); register-only controllers never emit XML.
- Two-phase dump: dirty-reset flush `0701000000600000000000000` (reg 06, uid 00000) then unit-scoped flush.
- Steady state: ackCAN 1/0 by CRC; `getCAN 0` nack → ack + resend; skip empty `setCAN` while `CAN2 in use`.
- **No synthesis**: reg-05-from-zones synthesis removed.
- Multi-unit: bank + projection handle all unit types (07 wired, 08 RF-connected); unit addressing end-to-end.

## Phases & tasks

| # | Issue | Phase | P | Depends on |
|---|-------|-------|---|------------|
| G-1 | Protocol message types + DTOs + addressing | contract | P0 | — |
| G-2 | Multi-consumer session layer | broker | P0 | G-1 |
| G-3 | Multi-unit register projection | broker | P0 | G-1 |
| G-4 | Typed decode/encode all registers + write-policy metadata | registers | P0 | G-1 |
| G-5 | `read` verb | broker | P0 | G-2, G-3, G-4 |
| G-6 | Write-policy enforcement + validation ranges | registers | P0 | G-4 |
| G-7 | JZ18 handshake | session | P1 | G-3 |
| G-8 | `status` frames | broker | P1 | G-2 |
| G-9 | Remove legacy/synthesis | cleanup | P1 | G-1, G-2 |
| G-10 | Broker-protocol acceptance tests | verify | P0 | G-2, G-5, G-6, G-7, G-8, G-9 |

## Critical path

```
G-1 → G-2 → G-5 → G-10
  └→ G-4 → G-6 → G-10
  └→ G-3 → G-7 → G-10
G-2 → G-8 → G-10
G-1 → G-2 → G-9 → G-10
```

Earliest completion: G-1 → G-2 → G-5 → G-10, with G-4/G-6 and G-3/G-7 landing in parallel before G-10.

## Cross-repo readiness gate

The sibling controller epic (aegis-controller#1) consumes this protocol. Its bridge issues are blocked by this epic's G-1 (the protocol contract) and must be end-to-end verified against this epic's G-10 acceptance tests. Cross-repo dependencies are recorded as native `blocked_by` links (globally-unique database ids) plus markdown hyperlinks.

## Out of scope

- The controller bridge (sibling epic).
- Multi-unit client handling in the controller (gateway supports it; bridge focuses on the primary unit).
- The legacy XML command channel (removed, not implemented).
- Raw bus-frame streaming to clients (removed; the controller re-encodes from typed data).

## Acceptance criteria (Done when)

- [ ] Any number of clients connect concurrently; each gets a full snapshot on connect and every subsequent `event`.
- [ ] A write from one client is echoed as an `event` to all clients (including the writer).
- [ ] All registers are typed and raw-addressable; write-policy violations return error acks.
- [ ] `read` returns `read_result` for any register.
- [ ] `status` frames reflect gateway/bus health.
- [ ] `cargo test` green; acceptance tests cover multi-client, broadcast, read, write-policy, batch acks, status, multi-unit.
