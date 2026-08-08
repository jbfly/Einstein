# iOS Einstein network egress — debug findings

## Symptom
Egg Freckles on the iPad (Einstein iOS build) reaches "connecting, will send" but
no data reaches the LAN server (`server.py` on mars, `192.168.100.93:6801`).
User sees the send flake: ~5 taps, ~4 NIE errors, then "connecting, will send".

## Cross-checks (rule out NewtonOS/NIE)
- Egg Freckles works on the **real Newton**.
- Egg Freckles works in the **Linux Einstein emulator** (FLTK build).
=> NewtonOS + NIE + NE2K driver + egg-freckles are all capable of transmitting.
   The fault is specific to the **iOS Einstein host build**.

## Debug instrumentation
Branch `ios-egress-debug`, commit `bd05b644` (built on 314668ab, ee92ac67 base).
`EINEGRESS` traces in `Emulator/Network/TUsermodeNetwork.cpp` at: ctor, frame-in
(SendPacket), DHCP lease, socket(), connect(), SO_ERROR.
KEY GOTCHA: `syslog(3)`/ASL does NOT surface in `idevicesyslog` on this iOS.
Must log via `os_log` (Apple) — that is what finally produced output.

## Evidence (idevicesyslog, 2026-08-07 15:03)
- `EINEGRESS ctor TUsermodeNetwork constructed` fired x2 on launch (pid 1043).
- ZERO `frame` / `dhcp` / `socket` / `connect` / `SO_ERROR` across a fresh boot
  AND repeated send taps. `grep -c EINEGRESS.*(frame|dhcp|socket|connect)` = 0.
- Raw evidence: `/home/jbfly/einstein-egress-evidence/ios-egress-ctoronly-2026-08-07.txt`

## Conclusion
The host-side `TUsermodeNetwork` backend is alive but receives **no outbound
Ethernet frame** from the emulated NE2000. Because the identical Newton stack
transmits under the Linux FLTK build, the gap is in the **iOS build's path from
the emulated NE2000 chip's TX to `TUsermodeNetwork::SendPacket`** (wiring /
NE2000 emulation enablement / interrupt scheduling), not in NewtonOS.

## Next
1. Diff the network + NE2000 TX wiring between the working `app/FLTK/TFLApp.cpp`
   and the iOS `app/iEinstein/Classes/iEinsteinViewController.mm` setup (how the
   network manager is attached to the emulator/NE2000 TX + interrupts).
2. Instrument the NE2000 TX native primitive (above TUsermodeNetwork) to confirm
   whether the Newton driver's TX even reaches the emulated chip on iOS.
