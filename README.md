# wine-macos-native

Experimental patches that run Windows programs on Apple Silicon Macs with a
**native arm64 Wine** – no Rosetta 2 – using FEX-Emu as the in-process
x86-64 → ARM64 translator (ARM64EC).

Status: experimental. Tested on macOS 26.6.2, Apple M2.

| Works | Does not work yet |
|---|---|
| ARM64 Windows programs (`cmd`, `notepad`) | 32-bit x86 programs (work in progress, see `docs/32bit-design.md`) |
| x86-64 programs through FEX (tested: FreePiano 2.2 with audio) | |

## Lineage

```
WineHQ Wine (LGPL-2.1-or-later)
 └─ citi94/wine-macos-arm64, branch macos-arm64-port (Wine 11.10 + macOS arm64 port)
     └─ this project: patches/wine/
FEX-Emu/FEX (MIT), tag FEX-2609 (395b132f346b1a45def246d10c52245edba1ef02)
 └─ this project: patches/fex/
llvm-mingw 20260922 (mingw-w64 headers)
 └─ this project: patches/llvm-mingw/
```

## What the patches change

* **TEB without x18.** macOS zeroes x18 on every return to user mode
  (interrupts, page faults, signals), so PE code cannot keep the TEB there.
  The TEB lives in pthread TSD slot 700 and `NtCurrentTeb()` (Wine, mingw
  headers, FEX JIT/dispatcher) reads it through `TPIDRRO_EL0`.
* The x18 repair heuristics of the base port are narrowed so they no longer
  fire on unrelated faults; the x16-clobbering resume trampoline is only a
  last-resort fallback.
* `WINEARM64ECEMU=1` lets wineserver accept AMD64 images when an in-process
  ARM64EC emulator (FEX `libarm64ecfex.dll`) is configured in
  `HKLM\Software\Microsoft\Wow64\amd64`.
* `KUSER_SHARED_DATA` relocation (0x17ffe0000) also applies to ARM64EC code.
* CPU feature flags are read from `sysctl hw.optional.*` on macOS.
* Build fix: configure with `ac_cv_func_pipe2=no` when the SDK is newer than
  the running macOS.

## Licensing

* Wine and all changes to it: **LGPL-2.1-or-later** (`COPYING.LIB`).
* FEX and changes to it: **MIT** (`patches/fex/LICENSE-FEX-MIT`); bundled FEX
  dependencies keep their own licenses.
* mingw-w64 header changes: same terms as mingw-w64 (public domain / ZPL).
* This repository contains no Windows fonts, no third-party Windows
  applications and no Wine prefixes.

### AI-generated code notice

The patches in this repository were written with the help of an AI assistant
(Claude). FEX-Emu does not accept AI-generated contributions
(see its `CLAUDE.md`); **do not submit `patches/fex/` upstream**. Check
WineHQ's current contribution policy before proposing anything to Wine.
