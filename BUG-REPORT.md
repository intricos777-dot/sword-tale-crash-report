# Sword Tale: Lost Excalibur — ST-001d Crash Report

**Bug:** Native `EXCEPTION_ACCESS_VIOLATION` (`0xc0000005`) in the shipping
game executable at **RVA `0x60E63`**, causing a silent hard-death at the
intro→gameplay transition. No dump, no log, no crash reporter — the process
simply vanishes mid-mission.

**Reported:** 2026-09-21 · Steam App ID **3305630** · UE **4.18.3** (CL-3832480)
· Proton Experimental (Linux) and reproducible as silent death on every online
session.

---

## TL;DR for the developer

In `SwordTale-Win64-Shipping.exe`, the SHA-1 routine epilogue does:

```
140060e5a: f3 0f 7f 07        movdqu %xmm0,(%rdi)          ; store digest (unaligned)
140060e5e: 66 0f 7e 4f 10     movd   %xmm1,0x10(%rdi)      ; store tail (unaligned)
140060e63: 0f 28 70 b8        movaps -0x48(%rax),%xmm6     ; <<< CRASH
```

`movaps` requires the source operand to be 16-byte aligned. The caller hands a
pointer in `rax` that is **not 16-byte aligned** (and in the captured run,
`info[1] = 0xFFFFFFFFFFFFFFFF`, a near-NULL dereference), so the CPU raises
`#GP`, and Windows surfaces it as `0xc0000005`. The exception handler fails to
dispatch (`Exception frame is not in stack limits => unable to dispatch
exception`), so **no crash report is ever generated** and the game silently
dies.

**One-line fix:** replace the aligned load with an unaligned load —
`movaps xmm6, -0x48(%rax)` → `movdqu xmm6, -0x48(%rax)` — or ensure the buffer
at `rax-0x48` is 16-byte aligned and lifetime-valid before calling the routine.

---

## Full evidence

### The fatal exception (Proton log, online session, 2026-09-21)

```
err:seh:NtRaiseException Exception frame is not in stack limits => unable to dispatch exception.
err:seh:raise_exception code=c0000005 flags=0 addr=0x140060e63
  info[0]=0000000000000000            (read access)
  info[1]=FFFFFFFFFFFFFFFF            (invalid/near-NULL address)
```

Confirmed execution context: the fault happens on a background worker thread
right after the .NET CLR (`mscoree`, 434 trace lines) and `NETAPI32.dll` load
— i.e., during the **intro→gameplay handoff**, when the game's managed
component performs a network/telemetry step.

### Disassembly at the crash site (RVA 0x60E63)

```
140060dd0: 0f 6f c8          movq   %mm0,%mm1
140060dd3: 0f 3a cc c2 03    sha1rnds4 $0x3,%xmm2,%xmm0
140060dd8: 0f 38 c8 cc       sha1nexte %xmm4,%xmm1
140060ddc: 66 0f ef fd       pxor   %xmm5,%xmm7
140060de0: 0f 38 ca fe       sha1msg2 %xmm6,%xmm7
140060de4: f3 0f 6f 26       movdqu (%rsi),%xmm4
140060de8: 66 0f 6f d0       movdqa %xmm0,%xmm2
140060dec: 0f 3a cc c1 03    sha1rnds4 $0x3,%xmm1,%xmm0
140060df1: 0f 38 c8 d5       sha1nexte %xmm5,%xmm2
140060df5: f3 0f 6f 6e 10    movdqu 0x10(%rsi),%xmm5
140060dfa: 66 0f 38 00 e3    pshufb %xmm3,%xmm4
140060dff: 66 0f 6f c8       movdqa %xmm0,%xmm1
140060e03: 0f 3a cc c2 03    sha1rnds4 $0x3,%xmm2,%xmm0
140060e08: 0f 38 c8 ce       sha1nexte %xmm6,%xmm1
140060e0c: f3 0f 6f 76 20    movdqu 0x20(%rsi),%xmm6
140060e11: 66 0f 38 00 eb    pshufb %xmm3,%xmm5
140060e16: 66 0f 6f d0       movdqa %xmm0,%xmm2
140060e1a: 0f 3a cc c1 03    sha1rnds4 $0x3,%xmm1,%xmm0
140060e1f: 0f 38 c8 d7       sha1nexte %xmm7,%xmm2
140060e23: f3 0f 6f 7e 30    movdqu 0x30(%rsi),%xmm7
140060e28: 66 0f 38 00 f3    pshufb %xmm3,%xmm6
140060e2d: 66 0f 6f c8       movdqa %xmm0,%xmm1
140060e31: 0f 3a cc c2 03    sha1rnds4 $0x3,%xmm2,%xmm0
140060e36: 41 0f 38 c8 c9    sha1nexte %xmm9,%xmm1
140060e3b: 66 0f 38 00 fb    pshufb %xmm3,%xmm7
140060e40: 66 41 0f fe c0    paddd  %xmm8,%xmm0
140060e45: 66 44 0f 6f c9    movdqa %xmm1,%xmm9
140060e4a: 0f 85 f0 fd ff ff jne    0x140060c40
140060e50: 66 0f 70 c0 1b    pshufd $0x1b,%xmm0,%xmm0
140060e55: 66 0f 70 c9 1b    pshufd $0x1b,%xmm1,%xmm1
140060e5a: f3 0f 7f 07       movdqu %xmm0,(%rdi)
140060e5e: 66 0f 7e 4f 10    movd   %xmm1,0x10(%rdi)
140060e63: 0f 28 70 b8       movaps -0x48(%rax),%xmm6    ; CRASH HERE
```

This is a hand-rolled **Intel SHA-NI (SHA-1) implementation**. Notably, every
*input block* load in the loop uses `movdqu` (unaligned-safe), but this final
prologue/epilogue load of a working variable uses `movaps` (alignment-
required). The routine is fed a buffer pointer that is not correctly aligned
(and in the captured run, effectively null).

### Why no crash dump ever appears

```
Exception frame is not in stack limits => unable to dispatch exception
```

Wine/Windows cannot deliver the exception to the normal chain because the
faulting thread's stack frame cannot be walked/dispatched — the crash
reporter never runs. Hence: no `UE4CC-*` directory, no `Saved/Logs`, zero-byte
Breakpad asserts. Every "vanished corpse" report in the Steam reviews has the
same signature.

### Root-cause chain

1. The exe carries a **CLR runtime header** (mixed-mode native + .NET binary).
2. At the intro→gameplay transition, the managed component performs a
   **network/telemetry/Steam handshake** (NETAPI32 / netutils load observed
   immediately before the fault; `mscoree` traces throughout).
3. That handshake calls the game's native SHA-1 routine with a **bad*,
   misaligned or dangling buffer.
4. `movaps` faults on alignment → `0xc0000005` → undispatchable → silent death.

**Why users see it as "random":** alignment varies with allocator state, .NET
GC, and timing. It is probabilistic per session, which is why no RAM/VRAM/DXVK
tuning ever made it reproducible on demand.

---

## Reproduction

1. Windows 10/11 or Linux + Proton, game App 3305630.
2. Launch **online** (Steam connected). Play through the intro into gameplay.
3. Within the first gameplay minutes the process silently exits. No dump.
4. Linux capture (torch channel): `PROTON_LOG=1 PROTON_LOG_DIR=/tmp/protonlog`
   captures `Exception 0xc0000005 ... addr=0000000140060E63` as the last
   meaningful event before process removal.

**Verified workaround (until fixed):** launch Steam in **Offline mode**. The
network handshake is skipped, the SHA-1 path is never fed the bad buffer, and
the game reaches checkpoints with zero crashes (verified 2026-09-21).

---

## References

- `evidence/` — captured session logs and disassembly (offline verified run +
  crash-site disassembly).
- Full community investigation, mitigations and configs:
  https://github.com/intricos777-dot/sword-tale-bugfix
- Controller remap repo: https://github.com/intricos777-dot/sword-tale-controller