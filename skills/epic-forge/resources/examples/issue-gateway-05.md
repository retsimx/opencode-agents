# G-5: `read` verb

**Epic**: #1
**Phase**: broker

## Goal

Implement the `read` message: a client requests a register read, the gateway queues a unit-scoped flush, waits for the `getCAN`, and returns a correlated `read_result` with the typed (or raw) payload.

## Context

Today clients only get state via snapshot-on-connect and pushed events. The broker needs on-demand reads (the controller bridge uses them for reconciliation after reconnect; any client may want a specific register).

## Protocol contract (from epic #1)

- Client → server: `read { msg_id, unit_type?, unit_id?, register, zone? }`
- Server → client: `read_result { msg_id, unit_type, unit_id, register, zone?, payload }`
- Payload typed for known registers, raw hex for unknown.
- Read of a write-only register (09) → `ack {status:"error", reason:"register 09 is write-only"}`.
- Read of an internal register (07) → `ack {status:"error", reason:"register 07 is handled internally"}`.

## Out of scope

- Write-policy enforcement beyond the read-side rejections above (G-6), status frames (G-8), tests (G-10).

## Files to modify

- `crates/aegis-gateway/src/ws.rs` — handle `Read`; correlate with `read_result`.
- `crates/aegis-engine/src/session.rs` — add a unit-scoped flush request path that queues the flush and returns the resulting register value.
- `crates/aegis-engine/src/event.rs` — add an engine command/event for a targeted read.
- `crates/aegis-protocol/src/convert.rs` — encode the read result as typed DTO or raw hex.

## Acceptance criteria

- [ ] `read` of a known register returns a typed `read_result` with the current bank value (after a fresh unit-scoped flush + getCAN).
- [ ] `read` of an unknown register (16/17/1d/1e) returns raw hex.
- [ ] `read` of a write-only register (09) returns an error ack.
- [ ] `read` of the internal register (07) returns an error ack.
- [ ] `read` with `zone` on a zone-bearing register returns that zone's value.
- [ ] `cargo test` green.

## Dependencies

- **Depends on**: G-2, G-3, G-4.
- **Blocks**: G-10.

## Implementation notes

- The read must be correlated: the `read_result` carries the client's `msg_id`.
- The unit-scoped flush is a reg-06 write with the target unit's uid; the resulting `getCAN` contains the requested register. If the register is not in the response, return the current bank value (or an error ack if absent).
- Reads are serialized with writes (one TX per ping); a read may take a few ping cycles.
- Timeout: if no `getCAN` arrives within a bounded window (e.g. 5 s), return an error ack.

## Test plan

- Integration tests: read a known register → typed read_result; read unknown → raw hex; read write-only → error ack; read with zone → correct zone value; read timeout path.
