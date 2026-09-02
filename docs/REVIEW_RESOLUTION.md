# Review resolution addendum

- PR: #1
- Base: `main`
- Resolution scope: the six existing review threads listed below.
- This document is a normative design addendum. It records the accepted resolution contract; it does not claim that implementation or tests have already run.
- Bot review is not retriggered.

## PRRT_kwDOTNkKVM6OZ9D9 — LAN exposure default

Problem: binding to `0.0.0.0` exposes the canvas on every interface, not only the advertised LAN interface.

Resolution: the default server MUST bind the exact advertised LAN interface/address. Binding `0.0.0.0` MUST require an explicit opt-in flag, and startup MUST print the effective bind address and exposure. The advertised origin and bind address MUST be derived consistently.

Focused verification before resolving this thread: start with defaults and inspect the listening address; start with the explicit all-interface flag and verify the visible warning and intended bind.

## PRRT_kwDOTNkKVM6OZ9D_ — WebSocket Origin allowlist

Problem: a WebSocket upgrade can be initiated by an untrusted origin unless Origin is checked during the handshake.

Resolution: the WebSocket upgrade MUST validate `Origin` against an explicit allowlist derived from the advertised origin/host/port. Missing or untrusted origins MUST be rejected before session establishment. Any development-only origin MUST be an explicit configuration exception.

Focused verification before resolving this thread: attempt upgrades with the advertised origin, an untrusted origin, and no Origin header, and assert only the allowed case succeeds.

## PRRT_kwDOTNkKVM6OZ9EB — private placement acknowledgement

Problem: a sender needs authoritative acknowledgement and cooldown state; a broadcast pixel event is not sufficient sender feedback.

Resolution: an accepted `PLACE` MUST produce a sender-private ACK containing a placement sequence/token and remaining cooldown. Rejected placement MUST return a reason. The broadcast PIXEL event MUST NOT be the only source used by the sender to determine acceptance or cooldown.

Focused verification before resolving this thread: send accepted, cooldown, and invalid placements and assert private ACK/rejection messages are correlated to the sender request.

## PRRT_kwDOTNkKVM6OZ9EF — stale receiver after Lagged

Problem: a lagged stream can send a fresh snapshot followed by retained stale frames that roll the receiver backward.

Resolution: after a Lagged condition, the receiver MUST obtain a current snapshot, then subscribe to a fresh receiver/stream. It MUST discard retained frames from the old receiver and accept only frames from the fresh subscription after the snapshot boundary.

Focused verification before resolving this thread: force lag, apply snapshot plus retained frames, and assert no stale frame can roll back the visible board.

## PRRT_kwDOTNkKVM6OZ9EI — accepted placement durability

Problem: a dirty snapshot every ten seconds can lose pixels that were already acknowledged.

Resolution: accepted placements MUST be appended to a durable journal/WAL before ACK and broadcast, with atomic/replayable persistence appropriate to the storage system. Recovery MUST replay accepted entries. The contract’s loss objective is zero acknowledged placements; any weaker bound must be explicitly approved and documented.

Focused verification before resolving this thread: acknowledge placements, terminate before the periodic snapshot, recover, and assert every acknowledged placement is restored.

## PRRT_kwDOTNkKVM6OZ9EK — client-count performance claim

Problem: a 100-client claim for 256x256 does not automatically hold for larger canvases; a 1024 snapshot is materially larger.

Resolution: the 100-client/latency claim is limited to the default 256x256 canvas. Larger sizes MUST have an explicit size cap, memory formula, and benchmark, or be disabled. The acceptance matrix MUST measure the worst supported size rather than extrapolate from 256x256.

Focused verification before resolving this thread: load test 100 clients at 256x256 and separately test every enabled larger size with snapshot size, memory, and latency limits.

## Verification status

The checks above are required acceptance criteria for implementation. This addendum intentionally reports no test result and no implementation-complete status.