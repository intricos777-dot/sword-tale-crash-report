# Steam Community Submission — Sword Tale : Lost Excalibur (App 3305630)

> Developer: UTCC-DGS · Publisher: ZDKHub
> Paste the body below into BOTH:
>  1. Steam Community Hub → Discussions → New Discussion (for players + devs)
>     https://steamcommunity.com/app/3305630/discussions/
>  2. (If you have a support/contact channel from the store page: "Visit the
>     website" / Facebook) — same text works.
> Reports are public; the dedicated report repo (with evidence + fix guide)
> is linked so the developer can act without any back-and-forth.

---

## Title: [Bug] Game silently crashes a few minutes into gameplay — root cause found, easy fix (RVA 0x60E63)

### Summary

Since launch, the game **silently exits** a few minutes into gameplay (after
the intro, during the first missions) — no error, no crash dialog, no log.
Many reviews describe the same "it just closes" crash. We root-caused it and
can fix it.

### Root cause (developer, please read)

The process dies with a **native access violation** in the game's own
executable, `SwordTale-Win64-Shipping.exe`:

```
EXCEPTION_ACCESS_VIOLATION (0xc0000005)
  addr=0000000140060E63
  Exception frame is not in stack limits => unable to dispatch exception
```

At RVA **0x60E63** the disassembly is:

```
140060e63: movaps -0x48(%rax),%xmm6      ; CRASH — aligned SSE load
```

This sits at the end of a hand-rolled **Intel SHA-NI SHA-1 routine** (visible
SHA-NI instructions `sha1rnds4` / `sha1nexte` / `sha1msg2` / `sha1msg1`
immediately before it). Every *input-block* load in the loop correctly uses
`movdqu` (unaligned-safe), but one *working* load uses **`movaps`, which
requires 16-byte alignment**. The caller feeds it a buffer that is misaligned
or already freed (in our capture, the address was effectively null:
`info[1]=FFFFFFFFFFFFFFFF`).

Because the exception cannot be dispatched, **no crash reporter ever runs** —
that's why there is no dump, no log, and the game just "disappears". This also
explains why the timing looks random: it depends on allocator/GC state.

Context observed at crash time: the **.NET CLR** (`mscoree`) is loaded (the
exe is a mixed-mode native + .NET binary) and `NETAPI32.dll` loads immediately
before the fault — i.e. a managed network/telemetry step at the intro →
gameplay transition feeds the bad buffer into SHA-1.

### The fix (one line, plus good hygiene)

1. In the SHA-1 epilogue: `movaps xmm6,-0x48(%rax)` → `movdqu xmm6,-0x48(%rax)`
   (or in C: `_mm_load_ps` → `_mm_loadu_si128`).
2. Enforce 16-byte alignment (or a defensive `memcpy`) at the SHA-1 call site,
   especially for buffers coming from the managed/.NET side.
3. Add `assert(((uintptr_t)p & 0xF) == 0)` in dev builds.

Full step-by-step: https://github.com/intricos777-dot/sword-tale-crash-report/blob/main/FIX-GUIDE.md

### Verified workaround for players (until patched)

Play with Steam in **Offline mode**. The network handshake is skipped, the bad
SHA-1 path is never triggered — we reached checkpoints with zero crashes.

### Evidence & report

- https://github.com/intricos777-dot/sword-tale-crash-report (full crash
  report, disassembly, captured logs)
- Companion repo with Linux stability configs:
  https://github.com/intricos777-dot/sword-tale-bugfix

*Reported by the community. No game files reproduced — documentation only.*