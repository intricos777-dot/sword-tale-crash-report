# Sword Tale: Lost Excalibur — ST-001d Crash Report

**Public bug report + developer fix guide** for the silent mid-game crash in
*Sword Tale: Lost Excalibur* (Steam App **3305630**, UE 4.18.3).

**Status:** Root cause identified · workaround verified · awaiting developer
patch.

---

## The bug in one paragraph

The game's shipping executable has a **native access violation at RVA
`0x60E63`** — a `movaps` (16-byte-aligned SSE load) inside a hand-rolled
**Intel SHA-NI SHA-1 routine** that is fed a misaligned or dangling buffer by a
**.NET (CLR) managed network/telemetry handshake** at the intro→gameplay
transition. The exception is **undispatchable**, so the crash reporter never
runs: no dump, no log, no warning — the process simply vanishes mid-mission.
This matches the "crashes soon after I start playing" reports on Steam.

**Verified workaround (2026-09-21):** launching Steam in **Offline mode**
skips the network handshake that feeds the bad buffer, and the game reaches
checkpoints with zero crashes.

---

## Repository contents

| File | Purpose |
|------|---------|
| [`BUG-REPORT.md`](BUG-REPORT.md) | Full crash report: exception evidence, disassembly, root-cause chain, reproduction steps |
| [`FIX-GUIDE.md`](FIX-GUIDE.md) | Developer prescription: the exact instruction change (`movaps`→`movdqu`) + call-site alignment + verification checklist |
| [`evidence/`](evidence/) | Captured session logs (verified offline run) + crash-site disassembly |

## Related repositories

- **Community diagnosis, mitigations & Linux configs:**
  https://github.com/intricos777-dot/sword-tale-bugfix
- **Controller/gamepad remap (Xbox-style):**
  https://github.com/intricos777-dot/sword-tale-controller

---

## For players (until a patch ships)

1. **Play from Steam in Offline mode** — verified to eliminate the crash.
2. See the bugfix repo for launch options + engine tuning that also improve
   stability/FPS on 8 GB RAM / 4 GB VRAM systems.

## For the developer (UTCC-DGS / ZDKHub)

1. Read [`BUG-REPORT.md`](BUG-REPORT.md) and [`FIX-GUIDE.md`](FIX-GUIDE.md).
2. The fix is a one-liner (aligned `movaps` → unaligned `movdqu`) plus sane
   alignment at the SHA-1 call site.
3. Reproduce: launch online, play past the intro, watch the process exit
   silently within minutes; confirm with `PROTON_LOG` on Linux.

## License

MIT — documentation and analysis only. **No game files are redistributed.**

---

*Reported by the community. Evidence-based, reproducible, and written for the
developer who can fix it in minutes.*