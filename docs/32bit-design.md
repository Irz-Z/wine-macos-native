# 32-bit x86 (WoW64) on native arm64 macOS — design notes

## Problem
macOS arm64 processes cannot map anything below 4 GB (enforced 4 GB
`__PAGEZERO`; `mmap(MAP_FIXED)` and `mach_vm_allocate` below 4 GB fail).
32-bit Windows code needs all of its memory below 4 GB.

## Approach: guest address base
Reserve a contiguous 4 GB window `BASE .. BASE+4GB` above 4 GB.
A 32-bit guest address `A` lives at host address `BASE + A`.

1. FEX (WoW64 JIT): add `BASE` to every guest memory access, instruction
   fetch and stack operation. FEX already adds segment bases in 32-bit mode
   (`OpDispatchBuilder::AppendSegmentOffset` / `GetSegment`); start there.
2. Wine WoW64 thunks: every 32↔64-bit pointer conversion
   (`ULongToPtr`, `PtrToUlong`, structure thunks) adds/subtracts `BASE`.
3. Wine virtual memory: allocations for WoW64 processes are confined to
   the window and reported to the guest as `addr - BASE`.

## Milestones
1. CRT-free 32-bit program returns an exit code.
2. 32-bit program with CRT prints text.
3. OrCAD PSpice 9.1 demo starts.

## FEX implementation note
In 32-bit guest mode FEX computes guest addresses in 32-bit (W) registers and
uses them directly as host pointers. ARM64's `[Xbase, Wn, UXTW]` addressing
adds a 64-bit base to a zero-extended 32-bit address in one instruction, so a
reserved GUEST_BASE register can be applied in the JIT backend
(`JIT/MemoryOps.cpp`: `GenerateMemOperand` / `ApplyMemOperand`) for all
guest memory IR ops (LoadMem, StoreMem, LoadMemTSO, ...). Atomics only take
`[Xn]` and need one extra `add`. Guest code fetch by RIP must also add the base.
