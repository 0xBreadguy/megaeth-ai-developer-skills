# Findings

## MegaEVM / Spec Review
- 2026-05-18: Local `knowledge/github/mega-evm` updated from `87503c7` to `12c0e07`.
- Major visible areas of change include REX5 docs, SequencerRegistry system contract, Oracle v2.0.0, keyless deploy handling, and mega-evme provider/replay changes.
- Highest-priority repo drift found in `megaeth-skills`: overconfident timing language (`10ms`, `<10ms`, `instant`) and lack of spec-version / REX5 awareness.
- Builder-facing guidance should distinguish protocol-level real-time properties from end-to-end RPC latency and should not imply unstable REX5 features are universally active.
