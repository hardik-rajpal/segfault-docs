# MIR combines in td

Idea 4, second half. A TableGen backend for combines on **real target
instructions**, post-instruction-selection.

## Status: the two halves are not the same

| | Declarative td support |
|---|---|
| **GMIR** combines — pre-ISel, generic `G_*` opcodes | Exists, mature |
| **MIR** combines — post-ISel, target opcodes | **Nothing. All handwritten C++** |

The GMIR half is done: `GICombineRule` (`llvm/include/llvm/Target/GlobalISel/Combine.td:72`),
declarative MIR patterns (`llvm/docs/GlobalISel/MIRPatterns.rst`),
`llvm/utils/TableGen/GlobalISelCombinerEmitter.cpp`, and per-target
`*Combine.td` in AArch64, AMDGPU, RISCV, X86, Mips, SPIRV, WebAssembly.

The MIR half does not exist. AArch64's GISel combiners all run at or before
selection (`llvm/lib/Target/AArch64/AArch64TargetMachine.cpp:778-801`):

```
O0PreLegalizerCombiner -> PreLegalizerCombiner -> PostLegalizerCombiner -> PostLegalizerLowering
```

Searching for any combiner scheduled after ISel returns nothing.

Post-selection rewriting is instead:

| Mechanism | Pattern declaration |
|---|---|
| `llvm/lib/CodeGen/MachineCombiner.cpp` | C++ `enum MachineCombinerPattern : unsigned` (`llvm/include/llvm/CodeGen/MachineCombinerPattern.h`), virtual hook `getMachineCombinerPatterns()` (`TargetInstrInfo.h:1292`), replacement built by `genAlternativeCodeSequence()` |
| `llvm/lib/CodeGen/PeepholeOptimizer.cpp` | pure C++ |
| Per-target | `AArch64MIPeepholeOpt`, `SMEPeepholeOpt`, `PPCMIPeephole`, `PPCPreEmitPeephole`, `SIPreEmitPeephole`, `SIPeepholeSDWA`, `NVPTXPeephole`, `BPFIRPeephole` — all C++ |

The only `.td` mentions of these mechanisms anywhere in-tree are comments
deferring to the C++:

```
AArch64InstrFormats.td:2802:  // MADD/MSUB generation is decided by MachineCombiner.cpp
SystemZInstrInfo.td:640:      // by the PeepholeOptimizer via FoldImmediate.
```

## The smoking gun

`AArch64InstrInfo.cpp:7768` onward is a declarative pattern table written as
C++ function calls:

```cpp
setFound(AArch64::MADDWrrr, 1, AArch64::WZR, MCP::MULADDW_OP1);
setFound(AArch64::MADDWrrr, 2, AArch64::WZR, MCP::MULADDW_OP2);
setFound(AArch64::MADDXrrr, 1, AArch64::XZR, MCP::MULADDX_OP1);
setFound(AArch64::MADDXrrr, 2, AArch64::XZR, MCP::MULADDX_OP2);
setFound(AArch64::MADDWrrr, 2, AArch64::WZR, MCP::MULSUBW_OP2);
...
setVFound(AArch64::MULv8i8, 1, MCP::MULADDv8i8_OP1);
```

Each line is *(opcode, operand index, required physreg, pattern ID)*. That is a
match pattern, spelled in the least checkable way available. Note the
`_OP1`/`_OP2` duplication — that is hand-rolled commutation, which
`GICombineRule` already handles declaratively via `MaxPermutations`.

This table is the demo target: replace a slice of it with td rules and diff the
line counts.

## Scope

**Pre-RA only.** `MachineCombiner` and `PeepholeOptimizer` run on SSA MIR, so
def-use matching works. The `*PreEmitPeephole` passes run post-RA where there is
no SSA and physreg operands break def-use chains — explicitly out of scope.

**One target.** AArch64, because the `MADD`/`MSUB` family above gives a
self-contained, well-understood set of rules with an existing C++ baseline to
diff against.

## What is harder than the GMIR case

Target instructions carry baggage that generic `G_*` opcodes do not. This is the
real modelling work:

- **Physreg operands.** At MIR level a multiply *is* `MADDWrrr $a, $b, WZR`
  (`AArch64InstrInfo.td:2940`, `MulAccumWAlias` at `:3053`). Patterns must be
  able to require a specific physical register in an operand slot.
- **Implicit defs/uses**, including condition-flag defs (NZCV).
- **Subregister indices.**
- **Register classes** (`GPR32`, `GPR64`) instead of LLTs.

## Open design question: the cost model

`MachineCombiner` is not a rewriter, it is a profitability decision. It fires
only when a pattern improves critical-path depth or register pressure:

```cpp
enum class CombinerObjective {
  MustReduceDepth,
  MustReduceRegisterPressure,
  Default
};
```

A purely declarative rule cannot express "only if this shortens the dependency
chain." Options:

1. Keep a C++ profitability hook per rule — mirrors how `GICombineRule` allows
   inline C++ `[{ ... }]` predicates. Least new design, least novel.
2. Add a declarative objective annotation, e.g.
   `let Objective = MustReduceDepth;`, and have the emitter wire it to the
   existing depth/pressure machinery.

Option 2 is the more interesting result and the thing to aim for.

Corroborating signal that the current mechanism is straining: the hook signature
takes `SmallVectorImpl<unsigned> &Patterns` rather than the enum type,
specifically so targets can mint pattern IDs outside `MachineCombinerPattern`.
It wants to be generated.

## Strawman syntax

Crib `GICombineRule` wholesale; the shape carries over.

```tablegen
def madd_fold : MIRCombineRule<
  (defs root:$dst),
  (match (MADDWrrr $mul, $a, $b, WZR),     // i.e. a mul
         (ADDWrr $dst, $mul, $c)),
  (apply (MADDWrrr $dst, $a, $b, $c))> {
  let Objective = MustReduceDepth;
}
```

Compare against `MULADDW_OP1` + `MULADDW_OP2` plus their
`genAlternativeCodeSequence` arms.

## Files to crib from

- `llvm/utils/TableGen/GlobalISelCombinerEmitter.cpp` — emitter structure
- `llvm/docs/GlobalISel/MIRPatterns.rst` — operand syntax, type inference,
  naming rules, `GITypeOf`, `GIVariadic`
- `llvm/include/llvm/Target/GlobalISel/Combine.td` — `GICombineRule`,
  `GICombinePatFrag`, `GIDefMatchData`, `GICombineGroup`, `GICombiner`,
  builtins (`GIReplaceReg`, `GIEraseRoot`)

## Relation to idea 1

Independent, but they compose: rule families here duplicate the same way the
`GICombineRule` families do, so a higher-order rule template would collapse
these too. Pick one as the primary.
