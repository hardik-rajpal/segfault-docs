# Segfault Docs

Goal: extend tablegen

## Ideas 
1. Reflection in tablegen
    - i.e. pass td classes around as params, then `def : klassPassedThroughParam`
    - Demo that writes itself: a higher-order `class CombineRuleTemplate<...>`
    that collapses the duplicated `GICombineRule` families in
    `llvm/include/llvm/Target/GlobalISel/Combine.td`.
        - `!foreach` can already abstract over *instructions*, because
        instructions are defs and defs are first-class:
        `!foreach(op, [G_SHL, G_ASHR, G_LSHR], (pattern (op $dst, $x, 0)))`
        - It can't reach the *rule structure*, so whole families are
        copy-paste: `match_selects`/`match_ands`/`match_ors`/`match_addos`
        are four identical lines with one opcode swapped; likewise
        `bitcast_bitcast_fold`/`fptrunc_fpext_fold`.
        - Concrete, visibly shorter output, and diffable against real
        upstream code.
2. Namespaces
3. Runtime manipulation of codegen without recompiling the compiler.
    - `TargetRegistry` is already a runtime registry; only the tblgen-generated
    `.inc` tables are frozen. Make one loadable — combine rules first.
    - Upstream already ships `-disable-rule`/`-only-enable-rule`. There is no
    `-add-rule`. That's the gap.
    - [Runtime codegen without recompiling](runtime-codegen-no-recompile/index.md)
4. Td backend for writing
    - ~~GMIR combines~~
        - G_SELECT (G_ICMP ne s1 x, 0) 1, 0 → x
        - seems to be done already.
    - or MIR combines
        - pick a target for this.
    - [MIR combines in td ](mir-combines-in-td/index.md)
5. multi-def patterns for GISel/Sdag pipelines.
    - add with carry patterns exist but
    this is about allowing generic multi-def patterns.
6. Delegation (kotlin) or composition in tablegen
7. Syntax to enable 
    ```
    let mayStore = 1 in defm : multiclassWithRecordsThatDontHaveMayStoreField
    ```

## Implemented

- TODO