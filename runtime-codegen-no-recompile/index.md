# Runtime codegen manipulation without recompiling the compiler

Idea 3. Load codegen rules at runtime instead of baking them into the binary.

## Thesis

`TargetRegistry` is *already* a runtime registry — targets self-register via
`LLVMInitializeFooTarget()` (`llvm/include/llvm/MC/TargetRegistry.h:770,1064`).
The only frozen part is the tblgen-generated `.inc` tables, because they are
`#include`d into a translation unit and compiled in.

So: make one table loadable at runtime. Combine rules first — smallest surface,
most visible payoff.

**TableGen is the reason LLVM has no backend plugins.**

## Upstream already ships half of this

The combiner emitter generates, for every combiner
(`llvm/utils/TableGen/GlobalISelCombinerEmitter.cpp:2535,2545`):

```
<combiner>-disable-rule
<combiner>-only-enable-rule
```

wired through a `RuleConfig` with `parseCommandLineOption()`
(`AArch64PreLegalizerCombiner.cpp:856`).

Runtime per-rule codegen control for a given target is therefore already
accepted upstream as legitimate. **What exists is subtractive. There is no
`-add-rule`.** That is the gap.

## Why change codegen for a target upstream already supports?

Main has all the *general* improvements. It declines correct-but-narrow rules on
maintenance grounds, so a rule that only helps one vendor's DSP loop cannot go
upstream by construction.

- **Version pinning.** Certified / safety-critical toolchains sit on an old
  release. A rule file delivers one fix without requalifying the toolchain.
- **Autotuning.** Searching for the best rule set inherently needs many
  variants of one target.
- **Proof of demand.** Every major vendor already ships a forked LLVM carrying
  codegen deltas for the same targets. The demand is settled; the only question
  is whether the delta has to be a fork.
- **Dev loop.** Independent of the above: `Combine.td` is 2680 lines shared by
  every target, and touching it re-runs tblgen, recompiles and relinks.

Concede: for general code on a well-supported target upstream really is better,
and "users tune their own compiler" is a bad pitch. Keep the framing at
dev/experimentation and vendor-delta.

## Plan

- **Do:** `llc -load-combines=rules.td` — edit a rule, rerun, codegen changes,
  no rebuild.
- **How:** interpret, don't JIT. MLIR's PDL already proves the
  bytecode-interpreter design (`mlir/lib/Rewrite/ByteCode.h`: *"a byte-code and
  interpreter for pattern rewrites"*). Far less work than generating and
  JIT-compiling C++.
- **Scope:** dev/experimentation path. AOT stays production — interpreted
  tables are slower, which is the whole point of TableGen. This mirrors IRDL's
  stated principle: runtime loading without giving up ahead-of-time
  compilation.

## Risks

- **Unverified rules miscompile silently**, with no build or test gate in the
  way. The strongest objection is support burden, not feasibility. Wants rule
  verification or a `-verify-combines` mode.
- **ABI fragility.** `LLVM_PLUGIN_API_VERSION` is a blunt integer guard and
  plugins are already tied to an exact LLVM build; backend interfaces churn
  more than `PassBuilder`.
- **Not universally available.** `LLVM_ENABLE_PLUGINS` defaults to
  `LLVM_ENABLE_PIC` (`llvm/CMakeLists.txt:1110`) — off in static and most
  Windows configs.

## Related plugin surface

| Layer | Status |
|---|---|
| LLVM middle-end passes | shipped (`llvm/include/llvm/Plugins/PassPlugin.h`) |
| LLVM codegen | one all-or-nothing hook: `PreCodeGenCallback` can *replace* codegen, not extend it |
| LLVM target tables | nothing — no `TargetPlugin` anywhere in tree |
| MLIR dialects + passes | shipped (`mlir/include/mlir/Tools/Plugins/`), plus runtime-defined ops via `ExtensibleDialect.h` |
