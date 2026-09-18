# Segfault Docs

Goal: extend tablegen

## What we built

**"template class"** — late-bound template parameters for TableGen.

TableGen already has template parameters, but three syntactic positions
refuse to take one: the **base class** in an inheritance list, the
**`bits<n>` width**, and **`Inst{hi-lo}` bit ranges**. All three verified
blocked. `template class` fixes all three by storing a templated class's
body unparsed at definition and monomorphising it fresh per distinct
argument tuple at each use, entirely inside the parser.

Demonstrated on SystemZ's 30 near-identical `.insn` directive classes,
collapsing them to 3 (+2 outliers): 258 lines to 63, every one of the 15
generated `.inc` files byte-identical, all 447 existing TableGen tests
still passing.

Started as "reflection in TableGen" — pass `.td` classes around as values,
then `def : klassPassedThroughParam`. The first concrete demo target
(collapsing the duplicated `GICombineRule` families in `Combine.td`) turned
out to be **disproved**: an ordinary multiclass already collapses that
family, verified with tblgen. The demo moved to the SystemZ `.insn`
classes, where the varying axis really is the base class — that's what
stuck, and what the full writeup below covers.

Full design notes, the ideas we tried and rejected along the way, the
correctness gate, and what's still open: [Late-bound template
parameters](reflection-in-tablegen/index.md)

## Other ideas we didn't pursue

Namespaces, runtime-loadable combine rules without recompiling, a MIR
combine-rule backend, multi-def GISel/SDAG patterns, delegation/composition
syntax, and a `let mayStore = 1 in defm : ...`-style field-injection syntax
all came up during ideation. None were built for this hackathon — trimmed
from this repo to keep it focused on what actually shipped.
