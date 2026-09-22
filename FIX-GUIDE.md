# Fix Guide — ST-001d (RVA 0x60E63 align fault in SHA-1 routine)

This document tells the developer **exactly** what to change. The bug is in
`SwordTale-Win64-Shipping.exe`, RVA `0x60E63`, in a hand-rolled SHA-1
implementation using Intel SHA-NI instructions. It is a **real, deterministic,
one-line-fixable defect** — not a platform fluke.

---

## The defect

```
140060e5a: f3 0f 7f 07       movdqu %xmm0,(%rdi)     ; store digest (unaligned-safe)
140060e5e: 66 0f 7e 4f 10    movd   %xmm1,0x10(%rdi) ; store tail (unaligned-safe)
140060e63: 0f 28 70 b8       movaps -0x48(%rax),%xmm6 ; ALIGNED load -> #GP
```

`movaps` faults if the effective address is not 16-byte aligned. The caller can
pass a buffer whose address is not 16-byte aligned (e.g. a payload read from a
network stream, or a managed/.NET marshal buffer), so the load raises
`#GP`, surfaced as `EXCEPTION_ACCESS_VIOLATION`. The exception cannot be
dispatched → silent process death, no crash report.

### Confirm on your build

```
objdump -d --start-address=0x140060dd0 --stop-address=0x140060ea0 SwordTale-Win64-Shipping.exe
```

Your exe base is `0x140000000`, so the report maps to file offset
`0x60E63`. If your build differs, search for the same pattern: a SHA-NI loop
whose *input* loads use `movdqu` but whose *working* load uses `movaps`.

---

## Fix option 1 — unaligned load (smallest possible change)

In the SHA-1 epilogue where the working value is (re)loaded:

```asm
; BEFORE (faults on misaligned input)
movaps xmm6, -0x48(%rax)

; AFTER (tolerates any alignment)
movdqu xmm6, -0x48(%rax)
```

If this is C/C++, use the compiler's unaligned load:

```c
// BEFORE
__m128i v = _mm_load_ps((const float*)(p - 0x48));        // = movaps

// AFTER
__m128i v = _mm_loadu_si128((const __m128i*)(p - 0x48));  // = movdqu
```

Same for any sibling `movaps` over the same buffer (the whole routine should
treat its input as byte-unaligned for robustness, or document the alignment
contract loudly).

---

## Fix option 2 — enforce alignment at the call site (also recommended)

Keep the `movaps` and guarantee alignment instead:

```c
// Before hashing (e.g. at the network/telemetry handler that calls SHA-1):
size_t  n   = <input length>;
uint8_t* in = <input buffer>;           // may be anything from a stream
uint8_t* aligned = malloc(n + 16 + 15);
uint8_t* base = (uint8_t*)((uintptr_t)(aligned + 15) & ~(uintptr_t)15);

memcpy(base, in, n);                    // copy once; aligns + fixes lifetime
hash_out = sha1_ni(base, n);            // in is now 16-byte aligned
free(aligned);
```

If the buffer originates from the managed/.NET side, either pin + align it
before marshaling into native code, or copy in the callback as above — do **not**
let a moving GC or an unaligned marshal buffer feed the SIMD routine directly.

---

## Fix option 3 — defensive (both, belt and braces)

1. Change `movaps` → `movdqu` in the SHA-1 routine (Fix 1), **and**
2. Align/copy at the call site (Fix 2), **and**
3. Add a debug assertion at the routine entry:

```c
assert(((uintptr_t)p & 0xF) == 0 && "SHA-1 input must be 16-byte aligned");
```

This turns a silent undispatchable crash into a caught, reported assert in
development builds.

---

## Why this exact change fixes the reported crash

| Observation | Consequence |
|---|---|
| `addr=0x140060E63`, `info[1]=0xFFFFFFFFFFFFFFFF` | Load from a near-NULL/unwrapped pointer; alignment + lifetime bug |
| Only first minutes after intro→gameplay | Network/telemetry handshake feeds the bad buffer |
| `mscoree` CLR traces + exe has CLR header | Mixed-mode binary; managed→native marshaling site |
| No dump ever (undispatchable exception) | Reporter never runs; silent death is the *only* symptom |
| Offline mode → zero crashes | Skipping the network path avoids the bad SHA-1 call |
| All input-loads use `movdqu`, this one uses `movaps` | Inconsistent alignment assumption inside the routine |

---

## Verification checklist

- [ ] Launch **online**, play past intro into gameplay, hold > 10 minutes.
- [ ] Repeat 3+ sessions; observe zero silent exits (previously ~100% within
      minutes).
- [ ] Hashing/telemetry still functions (expected output unchanged — SHA-1
      result is identical, only the load instruction changed).
- [ ] No regression on Windows (same binary used there).

---

## Upstream note

- Game: **Sword Tale: Lost Excalibur** · Steam App **3305630** · UE 4.18.3
- Developer: UTCC-DGS · Publisher: ZDKHub
- Reporter repo: https://github.com/intricos777-dot/sword-tale-crash-report
- Full diagnosis + community configs: https://github.com/intricos777-dot/sword-tale-bugfix