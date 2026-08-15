# Segfault Docs

Goal: extend tablegen

## Ideas 
1. Reflection in tablegen
2. Namespaces
3. Runtime inputs, JIT tablegen + compilation.
    - Subject to usecase discovery
4. Td backend for writing GMIR combines
    - G_SELECT (G_ICMP ne s1 x, 0) 1, 0 → x
    - or MIR combines
        - pick a target for this.
5. multi-def patterns for GISel/Sdag pipelines.
    - add with carry patterns exist but
    this is about allowing generic multi-def patterns.


## Implemented

- TODO