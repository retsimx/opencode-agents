# Worked Example (Generic / Fictional)

This directory is the **calibration target** for epic-forge output. It demonstrates the
exact level of detail and structure expected — but it is **fictional**. Nothing here
references any real project, product, brand, or repository.

The example is a fictional **"Aegis Climate Gateway"** — a two-repo system:

- `aegis-gateway` — a daemon that brokers typed registers to multiple clients over
  WebSocket (the "register broker").
- `aegis-controller` — an app that bridges legacy panel clients to the broker.

Use this example to match the density and format of epic/issue bodies. The names,
registers, unit ids, and clients are invented.

Files:
- `epic-gateway.md` — the fictional gateway epic (full dfn-grade body).
- `issue-gateway-01.md` — a fictional issue (full dfn-grade body).
- `issue-gateway-05.md` — a second fictional issue showing a depends-on chain
  (blocks G-10, read-verb semantics); the cross-repo dependency pattern lives in
  the epic's sibling reference (`epic-gateway.md`).
