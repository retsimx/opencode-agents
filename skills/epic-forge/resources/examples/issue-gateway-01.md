# G-1: Protocol message types + DTOs + addressing

**Epic**: #1
**Phase**: contract

## Goal

Define the new protocol message types, the typed DTOs for every register, and the addressing model in the gateway's protocol crate. This is the contract every other issue in this epic verifies against. No versioning — these types replace the existing `mailbox_*`/`write_can`/`direct`/`raw_can` surface in place.

## Context

The current protocol crate defines client/server messages with `mailbox_update`, `command`, `write_can`, `direct`, `mailbox_snapshot`, `mailbox_event`, `raw_can`, `direct_reply`, `ack`, `error`. These are replaced by the generic broker surface. The DTOs currently cover only regs 01/03/05; this issue adds DTOs for all registers and the raw-hex payload form.

## Protocol contract (normative — from epic #1)

### Addressing

```
{ "unit_type": "07", "unit_id": "3f2a1", "register": "03", "zone": 1 }
```
- `zone` only for zone-bearing registers (03, 04).
- `unit_type`/`unit_id` optional (default primary unit).
- `payload` = typed object OR raw 14-char hex string.

### Client → server

| type | fields |
|---|---|
| `write` | `msg_id`, `unit_type?`, `unit_id?`, `register`, `zone?`, `payload` |
| `read` | `msg_id`, `unit_type?`, `unit_id?`, `register`, `zone?` |
| `command` | `msg_id`, `action` (`resync`\|`flush_unit`) |

### Server → client

| type | fields |
|---|---|
| `snapshot` | `units` (multi-unit bank) |
| `event` | `unit_type`, `unit_id`, `register`, `zone?`, `payload` |
| `read_result` | `msg_id`, `unit_type`, `unit_id`, `register`, `zone?`, `payload` |
| `ack` | `msg_id`, `status` (`success`\|`error`), `reason?` |
| `status` | `state` (`negotiating`\|`synced`\|`resyncing`\|`link_down`), `detail?` |
| `error` | `message`, `reason?` |

### DTOs (exact field names/types)

| Reg | DTO fields |
|---|---|
| 01 `zone_config` | `header: u8`, `total_zones: u8`, `constant_zones: u8`, `constant_zone_ids: [u8;3]`, `filter_clean_required: bool` |
| 02 `unit_activation` | `unit_type: String`, `activation_status: String`, `dict_fw_major: u8`, `dict_fw_minor: u8` |
| 03 `zone_state` | `open: bool`, `damper_pct: u8`, `sensor_type: String`, `target_temp_c: f64`, `measured_temp_c: f64` |
| 04 `zone_limits` | `min_damper: u8`, `max_damper: u8`, `motion_status: u8`, `motion_config: u8`, `zone_error: u8`, `rssi: u8` |
| 05 `system_status` | `power: String`, `mode: String`, `fan: String`, `target_temp_c: f64`, `myzone_id: u8`, `fresh_air: bool`, `rf_sys_id: u8` |
| 06 `firmware` | `fw_major: u8`, `fw_minor: u8`, `cb_type: u8`, `rf_fw_major: u8` |
| 08 `system_error` | `error_code: String` |
| 09 `activation_code` | `action: String`, `unlock_code: String` (4 hex), `activation_days: u8` |
| 0a `unit_announcement` | (empty) |
| 12 `sensor_pairing` | read: `sensor_uid: String` (6 hex), `pairing: bool`, `sensor_rev: u8`; write: `sensor_uid: String`, `zone: u8` |
| 26 `rf_device_pairing` | `pairing_control: u8`, `rf_device_type: u8`, `zone_channel: u8` |
| 27 `rf_device_calibration` | `calibration_control: u8`, `channel: u8`, `up_down_position: u8` |

Enums: `mode` = `cool|heat|vent|auto|dry|myauto`; `fan` = `off|low|medium|high|auto|auto_aa`; `power` = `on|off`; `sensor_type` = `no_sensor|rf|wired|rf2can_booster|rf_x`; `activation_status` = `no_code|code_enabled|expired`; `unit_type` = `aegis_mk1|aegis_mk2|aegis_mk3`; `action` = `set_code|unlock`.

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

## Out of scope

- Session/broker wiring (G-2), multi-unit projection (G-3), write-policy enforcement (G-6), read verb (G-5), status frames (G-8), legacy removal (G-9), tests (G-10). This issue defines types only.

## Files to modify

- `crates/aegis-protocol/src/message.rs` — replace the client/server message enums with the new catalog.
- `crates/aegis-protocol/src/dto.rs` — add all DTO structs above; keep snake_case serde.
- `crates/aegis-protocol/src/convert.rs` — DTO ↔ wire codec for all registers, plus raw-hex passthrough for unknown.
- `crates/aegis-protocol/src/lib.rs` — re-export the new types.
- `crates/aegis-protocol/src/error.rs` — extend the encode error enum for the new codec paths.

## Acceptance criteria

- [ ] Client message = `Write`/`Read`/`Command`; server message = `Snapshot`/`Event`/`ReadResult`/`Ack`/`Status`/`Error` (serde `type` tag, snake_case).
- [ ] Every register 01–0a, 12, 26, 27 has a typed DTO; unknown registers (16/17/1d/1e) round-trip as raw hex.
- [ ] `payload` accepts either the typed object or a 14-char hex string.
- [ ] Zone-bearing registers (03, 04) carry `zone` in the address, not the payload.
- [ ] DTO ↔ wire round-trips pass for every register (unit tests with the wire layouts above).
- [ ] Raw-hex passthrough: `"01010330000100"` ↔ reg-05 bytes `[0x01,0x01,0x03,0x30,0x00,0x01,0x00]`.
- [ ] `cargo test -p aegis-protocol` green.

## Dependencies

- **Depends on**: none (contract foundation).
- **Blocks**: G-2, G-3, G-4, G-9.

## Implementation notes

- Keep serde `#[serde(tag = "type", rename_all = "snake_case")]` for both enums.
- DTO field names are the wire JSON keys (snake_case), exactly as in the table.
- The `payload` field on `Write`/`Event`/`ReadResult` is `serde_json::Value`; typed DTOs serialize into it, raw hex is a JSON string.
- `zone` is `Option<u8>` on the message, `None` for non-zone registers.
- The codec must reject out-of-range values at encode time only where the write policy (G-6) requires; this issue implements the codec itself, not the policy.

## Test plan

- Unit tests in the protocol crate: one round-trip test per register (DTO → bytes → DTO), raw-hex round-trip for unknown registers, enum string ↔ byte mapping for every enum value, serde JSON shape tests (golden JSON for each message type).
