# Late-bound template parameters in TableGen

*(Idea 1. Page path kept as `reflection-in-tablegen/` for link stability.)*

Originally scoped as "reflection" — make classes first-class values. **Pivoted**
to a broader and better-framed feature: TableGen already has template
parameters; there are just three syntactic positions that refuse to take one.
Fix those.

All line/file references are against `../segfault-llvm-project`
(`0664fb13c0dd`). Everything marked *verified* was run against
`build/bin/llvm-tblgen`.

## The three fixed positions

```tablegen
class DirectiveInsnRIE<dag outs, dag ins, string asmstr, list<dag> pattern>
  : InstRIEd<0, outs, ins, asmstr, pattern> {   // (1) base class — fixed
  bits<48> enc;                                 // (2) width — fixed
  let Inst{47-40} = enc{47-40};                 // (3) bit range — fixed
  let Inst{7-0}   = enc{7-0};
}
```

Everything else on that line is already parameterizable. Those three are not.
Each is *verified* blocked:

| # | Position | Error today | What must become late-bound |
|---|---|---|---|
| 1 | base class | `Couldn't find class 'C'` | a record's **field set** |
| 2 | `bits<n>` | `expected integer in bits<n> type` | `BitsRecTy::Size` |
| 3 | `Inst{hi-lo}` | `expected integer or bitrange` | `SetValue`'s `ArrayRef<unsigned> BitList` |

**Correction to an earlier draft of this page.** It ranked position 2 as "by
far the hardest" and called it *dependent types*. That was wrong, and the wrong
word imported the wrong connotation — type-level computation, inference,
undecidability. None of that is involved. `bits<n>` is a **non-type template
parameter**, exactly `template <unsigned N> struct bitset`: monomorphised at
instantiation, concrete before any backend runs, no runtime anywhere.

Measured blast radius, smallest first:

| Position | Touches | Core call sites to guard |
|---|---|---|
| 3 | parser + `Init` | `SetValue`'s bit list, one parameter |
| 2 | `RecTy` + `RecordVal` | 25 `getNumBits()` in `lib/TableGen` (21 more in backends are post-instantiation, safe) |
| 1 | parser + `Init` + type system | changes what fields a record *has*, which everything else derives from |

So the honest ordering is **3 < 2 < 1**, the reverse of what this page first
claimed. Position 1 — the one the demo needs — is the widest change, not the
narrowest.

### A fifth position: parameterizing a parameter's own type

Raised while refining: the plan says a *class* can be a parameter, but never
says what happens when that parameter is used where a **type** is expected.
Two distinct cases hide under that, and they land on opposite sides of scope.

**5a. Class-valued parameter in type position, inside the body.** *Verified:*

```tablegen
class Base { int X = 1; }
class Wrap<Base C> { C field; }        // error: Couldn't find class 'C'
class Wrap2<Base C> { list<C> fields; } // error: Couldn't find class 'C'
```

That is **the same error from the same function** as position 1's core blocker.
`ParseType`'s `tgtok::Id` case calls `ParseClassID` (`TGParser.cpp:1152`), which
is exactly what `ParseSubClassReference` calls (`:805`). So the position-1 fix —
a scope fallback in `ParseClassID` that resolves an identifier bound to a
`ClassInit` — **covers this for free**, at both call sites and inside `list<>`,
with no extra code. It was an unstated consequence of the design, not a gap in
it. Add a test; add no implementation.

**5b. A parameter's type depending on an *earlier* parameter.** *Verified:*

```tablegen
class Wrap<int n, bits<n> enc> { ... }  // error: expected integer in bits<n> type
```

This one the design genuinely does not reach, and the reason is structural.
`ParseTemplateArgList` (`:3753`) runs **eagerly at definition time**, before any
instantiation exists, so `n` has no value when `bits<n>` is parsed. Monomorphising
the body does not help — the body is not where this lives.

Fixing it means capturing the *argument list* unparsed as well, then binding
left-to-right at each use: parse param 1's type, bind arg 1, parse param 2's type
under that binding, and so on. That is how C++ does it. The cost is a
chicken-and-egg at the use site — argument values must be parsed before their
parameter types are known, so they would have to be built as untyped `Init`s and
type-checked afterwards, which unpicks `CheckTemplateArgValues` and the arity
check that currently runs against a fully-parsed `Record`.

**Out of scope.** Not because it is uninteresting — it is the most genuinely
novel of the five — but because nothing in the demo needs it: all seven SystemZ
parameters have types (`class<InstSystemZ>`, `int`, `list<int>`, `dag`, `string`,
`list<dag>`) that are fixed and independent of each other.

### A fourth position, deliberately out of scope

And one more thing you cannot parameterize: the **argument list at a
subclass reference**. In `Fmt<0, outs, ins, asmstr, pattern>` the *base* can
become a parameter (position 1) but the *arity and shape of the call* cannot.
That is C++ parameter packs, and it is strictly the most expensive of the four:
it makes template-argument binding itself variadic, which every other position
assumes is fixed.

It buys the demo **nothing** — all 14 classes in the byte-identical group take
the same `<0, outs, ins, asmstr, pattern>` signature, and even the outliers
differ in body, not in base-class arity. Recorded here so it is a decision, not
an oversight. Not implemented.

These rankings assumed the *late-bound* implementation, and are superseded by
*Design: monomorphise on use* — under which the ordering inverts again:
position 3's range form costs nothing, position 2 costs ~10 lines, and position 1
is the only real work. The blast-radius table above is retained because it still
describes the fallback design.

## Thesis

TableGen already has one half of this and not the other.

- **Defs are first-class.** A `def` is a `Record`, a `Record` has a `DefInit`
  (`Record::getDefInit()`, `Record.h:1753`), and a `DefInit` is a value you can
  store in a field, put in a list, pass to a template arg, and iterate with
  `!foreach`. This is why `!foreach(op, [G_SHL, G_ASHR], ...)` works.
- **Classes are not.** A class is *also* a `Record` — `RK_Class` in the same
  `RecordKind` enum (`Record.h:1665`) — but there is no way to name one in
  value position. `ParseIDValue` looks in the local scope, then
  `Records.getGlobal()`, then errors (`TGParser.cpp:1193-1219`). It never
  consults `Records.getClass()`.

So the representation is already uniform; only the *surface language* draws the
line. That is why position 1 is plausible at all, and it generalises: positions
2 and 3 are the same shape — a value the parser insists on knowing early.

## The boundary, precisely

Every place the grammar demands a literal class name:

| Site | Function | Location |
|---|---|---|
| `def X : Base<...>` / `class X : Base<...>` | `ParseClassID` via `ParseSubClassReference` | `TGParser.cpp:755`, `:802` |
| `Base<...>` in value position (anonymous def) | inline `Records.getClass()` | `TGParser.cpp:3026` |
| `Base f;` field / template-arg type | `ParseType` | `TGParser.cpp:1152` |
| `defm : MC<...>` | `ParseMultiClassID` | `TGParser.cpp:782` |
| `multiclass X : MC<...>` | `ParseSubMultiClassReference` | `TGParser.cpp:846` |
| `!cast<B>`, `!isa<B>`, `!exists<B>`, `!instances<B>`, `!getdagop<B>` | `ParseOperatorType` | `TGParser.cpp:1240-1500` |

And one explicit refusal, which is the closest thing in-tree to a prior
decision on this:

```cpp
  if (Type->getRecTyKind() == RecTy::RecordRecTyKind)
    return Error(Loc, "cannot define type alias for class type '" + ...);
```
`ParseDeftype`, `TGParser.cpp:4098`. `deftype` binds a name to a type but
deliberately excludes class types.

## Evidence the feature is wanted

The workaround already in tree is **name-based reflection**: build a def's name
as a string, then `!cast` it back to a record.

```
$ grep -rho '!cast<[A-Za-z_0-9]*>' llvm/lib/Target/*/*.td | sort | uniq -c | sort -rn | head -3
   3558 !cast<Instruction>
    226 !cast<LAInst>
    204 !cast<Intrinsic>
$ grep -rc '!cast<' llvm/lib/Target/*/*.td | awk -F: '{s+=$2} END {print s}'
6427
```

6427 uses. Every `!cast<Instruction>(NAME # "_suffix")` is a value being passed
through a string because the language has no typed way to pass it. The
codebase reflects constantly; it just does so unchecked.

## Demo target: SystemZ `.insn` directive classes

`llvm/lib/Target/SystemZ/SystemZInstrFormats.td:1764-1993` — **30 classes,
230 lines**, every one of this exact shape:

```tablegen
class DirectiveInsnRIE<dag outs, dag ins, string asmstr, list<dag> pattern>
  : InstRIEd<0, outs, ins, asmstr, pattern> {
  bits<48> enc;
  let Inst{47-40} = enc{47-40};
  let Inst{7-0}   = enc{7-0};
}
```

Normalising the bodies gives **11 distinct shapes** across the 30 classes:

```
  14x  bits<48> enc; let Inst{47-40} = enc{47-40}; let Inst{7-0} = enc{7-0};
   4x  bits<32> enc; let Inst{31-24} = enc{31-24};
   3x  bits<32> enc; let Inst{31-16} = enc{31-16};
   2x  bits<48> enc; let Inst{47-32} = enc{47-32};
   1x  ... (7 more singletons)
```

The 14-way group is byte-identical except for the base class:

```
DirectiveInsnRIE : InstRIEd     DirectiveInsnRIS : InstRIS
DirectiveInsnRRS : InstRRS      DirectiveInsnRSY : InstRSYa
DirectiveInsnRXF : InstRXF      DirectiveInsnRXY : InstRXYa
DirectiveInsnSIY : InstSIY      DirectiveInsnVRI : InstVRIe
DirectiveInsnVRR : InstVRRc     DirectiveInsnVRS : InstVRSc
DirectiveInsnVRV : InstVRV      DirectiveInsnRSE : InstRSEa
DirectiveInsnVRX : InstVRX      DirectiveInsnVSI : InstVSI
```

They cannot be shared today, because *the varying axis is the base class* and
the base class is the one thing you cannot parameterize.

Each class has exactly one use, in `SystemZInstrInfo.td:2279-2392`:

```tablegen
def InsnRIE : DirectiveInsnRIE<(outs), (ins imm64zx48:$enc, ...), ".insn rie,...", []>;
```

So with a class-valued template argument, the format moves to the use site:

```tablegen
class DirectiveInsn48HL<class<InstSystemZ> Fmt, dag outs, dag ins,
                        string asmstr, list<dag> pattern>
  : Fmt<0, outs, ins, asmstr, pattern> {
  bits<48> enc;
  let Inst{47-40} = enc{47-40};
  let Inst{7-0}   = enc{7-0};
}

def InsnRIE : DirectiveInsn48HL<InstRIEd, (outs), (ins ...), ".insn rie,...", []>;
```

### How far each position gets you

Measured over the 30 classes (`widths: bits<16> x2, bits<32> x8, bits<48> x20`;
only two classes carry anything beyond the `enc`/`Inst` pattern —
`DirectiveInsnRIL` adds `string type`, `DirectiveInsnRXE` adds `let M3 = 0`):

| Positions fixed | Classes | Why |
|---|---|---|
| none (today) | **30** | — |
| 1 (base class) | **11** | one per distinct body shape; the 7 singletons save nothing |
| 1 + 3 (base + bit ranges) | **3** + 2 outliers | one per width; the slice list becomes a parameter |
| 1 + 3 + 2 (+ `bits<n>`) | **1** + 2 outliers | width becomes a parameter too |

**Position 2 — by far the hardest — buys exactly 3 classes → 1.** That is the
staging argument in one line. Do 1, then 3, then stop and reassess.

With 1 + 3 the whole block is three classes of this shape:

```tablegen
class DirectiveInsn48<class<InstSystemZ> Fmt, list<int> slice,
                      dag outs, dag ins, string asmstr, list<dag> pattern>
  : Fmt<0, outs, ins, asmstr, pattern> {
  bits<48> enc;
  let Inst{slice} = enc{slice};
}

def InsnRIE : DirectiveInsn48<InstRIEd, [47...40, 7...0], (outs), (ins ...), ...>;
```

One `let` covers every shape because **multiple ranges in a single `let` already
work** — *verified*:

```tablegen
let Inst{15-12,3-0} = enc{15-12,3-0};
  =>  bits<16> Inst = { enc{15}, enc{14}, enc{13}, enc{12}, ?, ?, ?, ?,
                        ?, ?, ?, ?, enc{3}, enc{2}, enc{1}, enc{0} };
```

So position 3 needs only that the *index list* be an `Init`, not new
per-range syntax.

The gate is that every generated `.inc` is byte-identical.

### Why not the `Combine.td` demo from the idea list

`match_selects`/`match_ands`/`match_ors`/`match_addos`
(`llvm/include/llvm/Target/GlobalISel/Combine.td:1888-1911`) differ in three
places: the opcode list, the C++ helper name inside `[{ }]`, and the def name.
None of those is a class.

**Verified:** an ordinary multiclass already collapses them. `[{ }]` is just a
`StringInit` with `SF_Code`, and `#` concatenation preserves that format
(`StringInit::determineFormat`, `Record.h:722`), so the helper name
interpolates; `!dag` builds the `(wip_match_opcode ...)` node from a list.

```tablegen
multiclass MatchHelper<list<Instruction> opcodes, string helper> {
  def NAME : GICombineRule<
    (defs root:$root, build_fn_matchinfo:$matchinfo),
    !con((match !dag(wip_match_opcode, opcodes, ?):$root),
         (match [{ return Helper.}] # helper # [{(*${root}, ${matchinfo}); }])),
    (apply [{ Helper.applyBuildFn(*${root}, ${matchinfo}); }])>;
}

defm match_selects : MatchHelper<[G_SELECT], "matchSelect">;
defm match_ands    : MatchHelper<[G_AND], "matchAnd">;
defm match_addos   : MatchHelper<[G_SADDO, G_UADDO], "matchAddOverflow">;
```

emits records byte-identical to the handwritten ones:

```
def match_addos {	// GICombineRule
  dag Action0 = (match (wip_match_opcode G_SADDO, G_UADDO):$root, [{ return Helper.matchAddOverflow(*${root}, ${matchinfo}); }]);
  dag Action1 = (apply [{ Helper.applyBuildFn(*${root}, ${matchinfo}); }]);
}
```

(One wrinkle found on the way: the names argument of `!dag` must be written `?`
for the whole list — `!listsplat(?, n)` is rejected as untyped and
`!listsplat<string>` is not valid syntax.)

So `Combine.td` demonstrates a missing *idiom*, not a missing *feature*. It is
still worth landing as a standalone cleanup patch — it is a real line-count win
and it buys credibility upstream — but it is not evidence for reflection. Do
not lead with it.

## Design: monomorphise on use

**The chosen architecture.** Not "make TableGen's types and inheritance
late-bound" (that was the earlier draft, kept as a rejected alternative below).
Instead: a templated class is never built as a `Record` at all. It is stored
unexpanded, and each *use* instantiates a fresh, fully concrete class — inside
TGParser, so the full evaluator and real source locations are available.

This is the preprocessor idea moved in-tree, which keeps its one big advantage
(everything downstream of parsing sees only ordinary concrete records) and drops
its three costs (no separate parser, no lost diagnostics, no build stage).

### Mechanism

**At `template class W<...> : Base { ... }`:**

1. Parse the template argument list normally (`ParseTemplateArgList`).
2. Do **not** parse the body. Record the lexer position of the base-class list
   and body, skip to the matching `}`.
3. Register a field-less **primary template** `Record` named `W` (`RK_Class`).
   This exists solely so `isSubClassOf("W")` keeps working — see cons.
4. Store the whole thing in a new `TemplateClasses` map, parallel to the
   existing `MultiClasses`.

**At each use — `def D : W<InstRIEd, 48, [47...40, 7...0]>`:**

1. Parse the arguments normally (`ParseTemplateArgValueList`).
2. Key on `(W, argument tuple)`; look up the instantiation cache.
3. On a miss: push a scope binding each argument as a scope variable, reposition
   the lexer to the saved body range, and run the **existing**
   `ParseSubClassReference` + `ParseObjectBody` into a fresh `Record` named
   `W$0` (`RK_Class`). Add the primary `W` as a superclass. Cache and register.
4. Call `AddSubClass(D, W$0)` — the unmodified code path.

Downstream of step 4 nothing knows templates exist.

### Why this makes the three positions nearly free

Because the arguments are bound as ordinary scope variables *before* the body is
parsed, the parser sees concrete values. Measured:

| Position | Under monomorphisation | Cost |
|---|---|---|
| 3. `Inst{hi-lo}` | **already works today** — verified | **zero** |
| 3b. `Inst{slice}`, list-valued | `ParseRangePiece` folds to a single `IntInit`; accept `ListInit` and splat | ~15 lines |
| 2. `bits<n>` | `ParseType` demands `tgtok::IntVal`; accept a value that folds to a positive int | ~10 lines |
| 1. base class | `ClassRecTy` + `ClassInit` + a scope fallback in `ParseClassID` | the real work |

Position 3 needing *nothing* is the load-bearing measurement. `ParseRangePiece`
already calls `ParseValue()` and only then requires an `IntInit`
(`TGParser.cpp:1025-1035`) — the restriction was never syntactic. Verified with
the same scope mechanism instantiation would use:

```tablegen
defvar hi = 15;  defvar lo = 12;
class Foo : Base { bits<16> enc; let Inst{hi-lo} = enc{hi-lo}; }
  =>  bits<16> Inst = { enc{15}, enc{14}, enc{13}, enc{12}, ?, ?, ... };
```

And **§4's deferred inheritance disappears entirely** — the hardest design on
this page. At instantiation the base class is known, so there is no parse-time
window to defer across and no bound to check against. §4a/§4b remain as
background on why the problem existed; they no longer describe work.

### Supported syntax

```tablegen
template class DirectiveInsn<class<InstSystemZ> Fmt, int width, list<int> slice,
                             dag outs, dag ins, string asmstr, list<dag> pattern>
  : Fmt<0, outs, ins, asmstr, pattern> {
  bits<width> enc;
  let Inst{slice} = enc{slice};
}

def InsnRIE : DirectiveInsn<InstRIEd, 48, [47...40, 7...0],
                            (outs), (ins imm64zx48:$enc, ...), ".insn rie,...", []>;
```

Five additions, and nothing else:

| Syntax | Meaning |
|---|---|
| `template class N<...> : B { }` | opt-in; body is stored unparsed and re-parsed per distinct argument tuple |
| `class` / `class<Base>` | a template-argument *type*; values are classes, optionally bounded |
| bare `Foo` in value position | a `ClassInit` — today this is `error: Variable not defined` |
| a class-valued parameter in *type* position (`C field;`, `list<C> f;`) | falls out of the `ParseClassID` fix for free — §5a |
| `bits<expr>` | `expr` must fold to a positive integer at instantiation |
| `X{expr}` | `expr` must fold to an integer or `list<int>` |

Explicitly **not** proposed: `template multiclass`, introspection operators
(`!fields`, `!templateargs`, `!superclasses`), inheriting from an *unbounded*
`class` parameter, variadic argument lists (§4th position), and parameter types
that depend on earlier parameters (§5b).

### Files touched

| File | Change | Size |
|---|---|---|
| `llvm/lib/TableGen/TGLexer.h` | `tgtok::Template`; lexer save/restore struct over `CurPtr`, `CurBuf`, `CurBuffer`, `TokStart`, `CurCode`, `PrepIncludeStack` | small |
| `llvm/lib/TableGen/TGLexer.cpp` | one `StringSwitch` case in `LexIdentifier` (`:422`) | trivial |
| `llvm/lib/TableGen/TGParser.h` | `struct TemplateClass`; `TemplateClasses` map; instantiation cache; declarations | small |
| `llvm/lib/TableGen/TGParser.cpp` | `ParseTemplateClass`, body-range capture, `instantiateTemplateClass`; hooks in `ParseObject`, `ParseClassID` (`:755`), `ParseSubClassReference` (`:802`), `ParseSimpleValue` (`:3026`), `ParseType` (`:1128`), `ParseRangePiece` (`:1025`) | **the bulk** |
| `llvm/include/llvm/TableGen/Record.h` | `ClassRecTy` (new `RecTyKind`), `ClassInit` (new `InitKind`) | position 1 only |
| `llvm/lib/TableGen/Record.cpp` | `get`/`Profile`/`getAsString`/`typeIsConvertibleTo` for the above | position 1 only |
| `llvm/lib/TableGen/JSONBackend.cpp`, `DetailedRecordsBackend.cpp` | print the new kinds | trivial |
| `llvm/test/TableGen/*.td` | new tests, `RUN: not llvm-tblgen` for errors | — |
| `llvm/docs/TableGen/ProgRef.rst` | document the five additions | — |

Eight files. Only `Record.h`/`Record.cpp` are deep, and only for position 1 —
so **positions 2 and 3 can land first, entirely inside TGParser/TGLexer**, with
no change to the record model at all. That is a much better first patch than the
earlier plan's ordering.

### Cons and open decisions

1. **Class identity is the main risk.** Backends call `isSubClassOf(` **453
   times** (`llvm/utils/TableGen/`). `W$0` is not `W`, exactly as `W<int>` is not
   `W<float>` in C++. *Decision: register a field-less primary-template record
   `W` and make every instantiation inherit it.* **Open:** `getSuperClasses()`
   (8 backend uses) will now surface `W$0` names — check whether any backend
   prints or matches on them.

2. **Re-parse cost.** Each distinct argument tuple re-parses the body. Cache
   keyed on the tuple; identical uses are free. Precedent: `defm` already
   re-resolves multiclass entries per use. **Open:** measure `.inc` generation
   time on X86 (89k `.td` lines) and AArch64 (119k) before and after.

3. **Diagnostics.** Re-lexing the *original* buffer keeps real source locations,
   which is the main reason to do this in-tree rather than as a preprocessor. But
   an error in a templated body needs an "instantiated from here" note. **Open:**
   does TableGen have a note mechanism, or only `PrintError`/`PrintWarning`?
   **Open:** dedupe errors that repeat once per instantiation, or let them
   repeat?

4. **`template` is a new reserved word.** *Verified safe in-tree:* all 91
   occurrences across LLVM's `.td` files are inside comments, never identifiers.
   But it is still a hard break for any out-of-tree `.td` using it as a name.

5. **Opt-in keyword vs. inference.** Recommend the keyword: strictly additive,
   makes the re-parse cost visible at the definition, and avoids speculative
   parsing to decide whether a class is templated. **Open:** upstream may prefer
   inference, which would be a significant redesign.

6. **Enclosing context leakage.** Instantiation happens mid-parse inside whatever
   `let ... in` / `foreach` / `multiclass` encloses the *use*. An instantiated
   class must not absorb the enclosing let-stack. **Open:** verify against
   `ApplyLetStack`; write a test where a `def` inside `let X = 1 in` uses a
   templated class.

7. **Recursion.** `template class A<...> : A<...>` needs a depth limit and a
   clear diagnostic. C++ has one for the same reason.

8. **Position 1 could be dodged.** Binding `Fmt` to a *string* and resolving by
   name is cheaper than `ClassRecTy`/`ClassInit` — but it reintroduces exactly
   the unchecked `!cast<Instruction>(NAME # ...)` idiom this whole idea exists to
   kill. *Recommend typed;* record the cheap option only as a fallback.

9. **Body capture mechanism.** Primary: save/restore lexer state and re-lex in
   place. Alternative: copy the body text into a new `SourceMgr` buffer per
   instantiation — simpler, but diagnostics then point into a synthetic buffer.
   **Open, leaning save/restore.**

### Rejected alternative: late-bound types and inheritance

The earlier draft kept one class `Record` with late-bound pieces: a `RecTy`
carrying an unresolved `Init` for `bits<n>`, an `Init` bit-list in `SetValue`,
and deferred `AddSubClass` with bounded class parameters (§4/§4b). It works, but
it is strictly more code in strictly more places — ~25 `getNumBits()` guards,
deferred type-compatibility checking, and a record whose field set is unknown at
parse time. Monomorphisation avoids all of it.

Its one genuine advantage: a single class `Record` preserves `isSubClassOf`
semantics for free, where monomorphisation needs the primary-template trick
(con 1). If that trick fails against real backends, this is the fallback.

## Blast radius

Monomorphisation keeps the record model almost untouched, because everything
downstream of `AddSubClass` sees ordinary concrete classes.

- **Stages 1-2 touch no record model at all** — `TGLexer.{h,cpp}` and
  `TGParser.{h,cpp}` only.
- **Stage 3** adds one `RecTyKind` and one `InitKind`. Those land in switches
  that are rarely exhaustive: 16 `getRecTyKind`/`RecordRecTyKind` sites and 6
  `IK_DefInit`/`IK_VarDefInit` sites across `llvm/`, plus printing in
  `JSONBackend.cpp` and `DetailedRecordsBackend.cpp`.
- **The 453 `isSubClassOf(` calls in `llvm/utils/TableGen/` are the real
  exposure**, and they are covered by the primary-template record rather than by
  editing any of them (con 1).

The earlier late-bound design also needed ~25 `getNumBits()` guards and deferred
type-compatibility checking. Monomorphisation needs neither.

## Implementation route: preprocessor vs. in-tree

Alternative proposal: don't touch TableGen. Write `tblgen-templated`, a
front-end that reads extended `.td`, monomorphises every template use, mangles
the instantiated names, and emits plain `.td.inc` that stock `llvm-tblgen`
consumes unchanged.

**The strongest argument for it is that monomorphisation erases all three hard
parts at once.** After expansion there is nothing late-bound left:

| Position | In-tree cost | Preprocessor cost |
|---|---|---|
| 1 base class | deferred inheritance + bounds (§4) — the hardest design on this page | `class W__InstRIEd : InstRIEd<...>` — textual |
| 2 `bits<n>` | `RecTy` + `RecordVal` surgery, 25 guarded sites | `bits<48>` — textual |
| 3 bit ranges | `SetValue`'s bit list becomes an `Init` | `Inst{47-40,7-0}` — textual |

§4b's "the bound is forced" argument also dissolves: there is no parse-time
window in which the base is unknown, so nothing needs checking against a bound.

**It works for the demo.** *Verified*: all 31 use sites are plain
`def X : Class<literal args>` inside one `let ... in { }` block
(`SystemZInstrInfo.td:2278-2392`) — no `foreach`, no `multiclass`, no `defm`,
no `!cast`, no `defvar`. A shallow parser is enough.

**Three costs, verified:**

1. **No line directives.** TableGen has a preprocessor — `#ifdef`, `#ifndef`,
   `#else`, `#endif`, `#define` (`TGLexer.h:67-73`) — but *no `#line`*. So
   every tblgen diagnostic points into generated `.td.inc` with no way to map
   back to the source. Note also what that directive set implies: TableGen's
   preprocessor deliberately has no function-like macros and no line control.
   That minimality looks like a design stance, not an oversight.

2. **Build integration.** `tablegen()` tracks `.td` include dependencies with
   `-d ${ofn}.d` / `DEPFILE` (`llvm/cmake/modules/TableGen.cmake:45-46`).
   Inserting a stage means reproducing that scanning or losing incremental
   builds.

3. **The scaling wall.** To expand a template you need *every use site*. In the
   demo they are literal. Everywhere else in LLVM they sit inside `foreach`,
   `multiclass`, `defm`, guarded by `#ifdef`, with arguments computed by
   `!cast`/`!if`/`defvar`. Handling those means reimplementing most of
   `TGParser` — at which point modifying TableGen was the cheaper path all
   along.

Secondary: `tblgen-lsp-server`, `--dump-json` and `-print-detailed-records` all
operate on real `.td`, so the extended language gets no tooling. And upstream
would reasonably answer "if this is good, put it in TableGen" — a preprocessor
is a fork wearing a hat.

### Verdict: not taken

Cost 3 — the scaling wall — is decisive. A preprocessor that handles the demo's
literal arguments looks finished, while the distance to LLVM's real `.td`
(`foreach`/`multiclass`/`defm`, `!cast`-computed arguments) is the entire
project. Reimplementing that much of `TGParser` is strictly worse than modifying
it.

**Decision: implement in-tree.** The good idea inside the preprocessor proposal
— monomorphise, so everything downstream sees only concrete records — is kept,
and moved into TGParser where the evaluator and source locations already exist.
See *Design: monomorphise on use*.

## Staging

Reordered for the monomorphisation architecture. The machinery lands first; the
three positions then fall out cheaply, deepest last.

| Stage | Content | Gate |
|---|---|---|
| 0 | ~~Build `llvm-tblgen`~~ **done**; capture baseline `.inc` for every SystemZ backend | hashes recorded |
| 1 | `template class` machinery: keyword, lexer save/restore, body capture, instantiate-on-use, cache, primary-template record | a `template class` whose args are used only in *ordinary* positions behaves identically to a plain class; `isSubClassOf` holds |
| 2 | Position 3b (list-valued `X{expr}`) then position 2 (`bits<expr>`) | ~25 lines total, both inside TGParser; position 3's range form needs nothing |
| 3 | Position 1: `ClassRecTy`, `ClassInit`, `ParseClassID` scope fallback | `template class W<class<B> C> : C<...>` |
| 4 | SystemZ demo: 30 classes → 3 + 2 outliers | every generated `.inc` byte-identical to stage 0 |
| 5 | Perf measurement (X86, AArch64), `ProgRef.rst`, upstream RFC | no significant `.inc` generation regression |

**Hackathon scope.** This is built for a demo, not for an RFC. The MVP path is
**1 → 3 → 2 → 4**: template machinery, class-valued parameters, the two cheap
positions, SystemZ collapse. Only **stage 5** (perf measurement, `ProgRef.rst`,
upstream RFC) is dropped.

*An earlier version of this note cut stage 2 as "only takes 3 classes to 1".
That was wrong — it attributed the whole stage to its weaker half.* Stage 2
bundles two independent changes with very different payoffs:

| Change | Cost | Demo effect |
|---|---|---|
| list-valued `X{slice}` (position 3b) | ~15 lines | **11 → 3 classes** |
| `bits<expr>` (position 2) | ~10 lines | 3 → 1 class |

~25 lines for 11 → 3 is the best effort-to-payoff ratio on the page. Both stay.

Correctness gate stays: all 11 SystemZ `.inc` hashes must match. Everything
else — recursion limits beyond a depth counter, note diagnostics, error dedup,
out-of-tree `template` collisions, `getSuperClasses()` name leakage — is
documented above and explicitly not handled.

**Decisions taken, so they stop being open:** typed `ClassInit` over the
string-based dodge (con 8); lexer save/restore over a copied buffer (con 9);
opt-in keyword over inference (con 5). The "no notes" call (con 3) is
**reversed** — `PrintNote` already exists, so notes are in.

Stage 1 is the risky one and the only one that needs new architecture. Stages 2
and 3 are additive and independently revertible. **Stage 4 is the deliverable.**

Note the reversal against the earlier plan: position 2 was staged *last* there
because late-bound `RecTy` was expensive. Under monomorphisation it is ~10 lines
and lands in stage 2.

Tests go in `llvm/test/TableGen/` (372 files there already, all
`llvm-tblgen | FileCheck`), with error cases as `RUN: not llvm-tblgen`.

## What was measured

`llvm-tblgen` built at `../segfault-llvm-project/build/bin/llvm-tblgen`
(Release + assertions, AArch64 only — enough, since tblgen reads any target's
`.td` with the right `-I` flags).

Each of these is a two-line `.td` file; all four confirm a boundary claimed
above.

**The core blocker** — inheriting from a class-valued parameter:

```tablegen
class Base<int x> { int X = x; }
class Wrap<Base C> : C<1> { }
```
```
e5.td:2:22: error: Couldn't find class 'C'
```

**A class name in value position**, and via `defvar`:

```tablegen
def D { Base b = Base; }      // error: Variable not defined: 'Base'
defvar C = Base;              // error: Variable not defined: 'Base'
```

Note this is `ParseIDValue`'s fall-through error, exactly as read at
`TGParser.cpp:1219` — nothing else claims the identifier, so §2 is a pure
extension.

**`deftype` refuses class types** (`TGParser.cpp:4098`):

```tablegen
deftype T = Base;   // error: cannot define type alias for class type 'Base'
```

**`bits<n>` and bit ranges must be literal integers:**

```tablegen
class Foo<int n> { bits<n> enc; }              // error: expected integer in bits<n> type
class Foo<int hi, int lo> : Base {
  let Inst{hi-lo} = enc{hi-lo};                // error: expected integer or bitrange
}
```

Consequence for the demo: the SystemZ block collapses to **four** shared
classes (one per distinct width/slice shape), not one. Fully collapsing it
would need computed `bits<n>` and computed bit ranges — a separate, larger
feature. Do not scope-creep into it.

**Multi-range `let` works; arithmetic on unresolved bits does not.** Both
shown in §4c — together they set the design for position 3.

**`foreach` does not work inside a class body:**

```tablegen
class Foo : Base { foreach i = [4,5,6,7] in let Inst{i} = 1; }
// error: Unknown token when expecting a type
```

This is why position 3 must be a list-valued index rather than a loop.

**Baseline harness works.** The demo gate is mechanical:

```
$ llvm-tblgen -gen-instr-info -I llvm/lib/Target/SystemZ -I llvm/include \
    -I llvm/lib/Target llvm/lib/Target/SystemZ/SystemZ.td -o sysz-instr.inc
$ wc -l sysz-instr.inc   # 15873
$ md5sum sysz-instr.inc  # c1c248adbbe5e34e95bb69299ee7a7f8
```

Capture this for every backend before touching the `.td`, and require an
identical hash after.

## Still to check

Ordered; each can change a decision in *Cons and open decisions*.

1. ~~Does TableGen have a diagnostic "note" mechanism?~~ **Answered — yes.**
   See *Resolved before implementation* §2.
2. ~~Does `ApplyLetStack` leak into an instantiated class?~~ **Answered — yes,
   and it is on the demo's critical path.** See §1.
3. ~~Do any backends read `getSuperClasses()` names?~~ **Answered — three do.**
   See §3.
4. **Re-parse cost on a large target.** X86 and AArch64 `.inc` generation time,
   before vs. after.
5. **Why did `deftype` exclude class types** (`TGParser.cpp:4098`)? Search the
   review history — still the strongest likely objection to position 1.

## Resolved before implementation

The three empirical items from *Still to check* were run against the tree. All
three came back with something that changes the implementation.

### 1. `let` leakage is real, and it is on the demo's critical path (con 6)

Not hypothetical. The 31 demo use sites all sit inside:

```tablegen
let isCodeGenOnly = 1, hasSideEffects = 1 in {      // SystemZInstrInfo.td:2278
  def InsnE : DirectiveInsnE<(outs), (ins imm64zx16:$enc), ".insn e,$enc", []>;
  ...
}
```

Instantiation runs mid-parse *inside* that block, and `ParseObjectBody` calls
`ApplyLetStack(CurRec)` unconditionally (`TGParser.cpp:3985`, applying the whole
`LetStack` at `:3922`). So the instantiated class absorbs `isCodeGenOnly = 1`.

For this demo it happens to be harmless — the same value is then set again on the
def. **The instantiation cache makes it harmful in general:** instantiate `W<A>`
inside a `let X = 1 in`, cache it, then use `W<A>` outside that `let`, and the
second use silently inherits `X = 1` from the first use's enclosing context. The
cache turns a local leak into action at a distance.

*Fix, decided:* instantiate in a pristine parser context — save and clear
`LetStack` (and `Loops`, and `CurMultiClass`) around the re-parse, restore
after. A templated class body must see only its own arguments. Cheap, and it has
to be in stage 1, not bolted on later.

### 2. TableGen *does* have a note mechanism (con 3)

`PrintNote` exists in three overloads, including `PrintNote(ArrayRef<SMLoc>,
const Twine &)` (`llvm/include/llvm/TableGen/Error.h:25-27`). The open question
assumed it might not.

This reverses the earlier decision to skip notes: "instantiated from here" is now
a few lines against an existing API rather than a new diagnostic feature, and it
is the difference between a usable error and a baffling one. *Do it.* Error
dedup across repeated instantiations stays out of scope.

### 3. Superclass names do leak, into three name-matching call sites (con 1)

`Record::getSuperClasses()` / `getDirectSuperClasses()` readers that match on
**name**, not identity:

| Site | What it does | Effect of a mangled `W$0` |
|---|---|---|
| `SearchableTableEmitter.cpp:1013` | `getDirectSuperClasses().size() != 1` | **worst case** — the primary-template trick adds a superclass, breaking an *arity* check |
| `CallingConvEmitter.cpp:116` | `starts_with("CCIfSwift")` on the name | misses unless the mangling preserves the prefix |
| `SetTheory.cpp:315` | looks up an expander by superclass name | `W$0` misses; the primary `W` is also in the list, so it still resolves |

`RegisterInfoEmitter`'s four `getSuperClasses()` calls are
`CodeGenRegisterClass::getSuperClasses()` — a different class, not `Record`'s.
Not exposure.

None of the three is reachable from the SystemZ demo, and all of them are
**self-detecting**: the gate is byte-identical `.inc` output, so a leak that
changed anything would fail the gate rather than ship. Recorded as a known
limitation of templating a `SearchableTable` or a `CCIf*` class, not handled.

## Built

Implemented on branch `tablegen-template-classes` in `../segfault-llvm-project`,
three commits on top of `0664fb13c0dd`. **All five stages of the MVP landed.**

| | |
|---|---|
| Gate | all 15 SystemZ `.inc` outputs unchanged |
| Result | **passed** — 14 byte-identical; `InstrInfo.inc` differs only in `SystemZInstrFormats.td:NNNN` source-line comments, which move because the file is 188 lines shorter. Masking those line numbers leaves it byte-identical too. |
| Tests | `llvm/test/TableGen` **447/447**, including a new `TemplateClass.td` |
| Demo | **30 classes → 3**; 258 lines → 63 across the two `.td` files |
| Feature size | ~500 lines across 6 source files |

### The collapsed block

```tablegen
template class DirectiveInsn<class<InstSystemZ> Fmt, int opcode, int width,
                             list<int> slice, dag outs, dag ins, string asmstr,
                             list<dag> pattern>
  : Fmt<opcode, outs, ins, asmstr, pattern> {
  bits<width> enc;
  let Inst{slice} = enc{slice};
}
```

`DirectiveInsnRIL` and `DirectiveInsnRXE` remain as two-line classes built on
the template, because they carry something beyond the shared shape (a
`string type` field and `let M3 = 0`). Everything else became a use-site
argument tuple.

### What the build changed about the design

1. **The slice list needs no new syntax.** The page proposed `[47...40, 7...0]`,
   which is not valid TableGen — `...` is range syntax inside `{ }`, not inside
   `[ ]`. `!listconcat(!range(40, 48), !range(0, 8))` expresses the same set
   today. *Verified:* `let Inst{s} = enc{s}` pairs the two sides element-wise, so
   only the set of indices matters and ascending order is free. One `defvar` per
   distinct shape carries it.

2. **Position 3b lands in one function, not two.** Both the assignment target
   (`Inst{slice}`) and the value suffix (`enc{slice}`) funnel through
   `ParseRangePiece`. Accepting a `ListInit` there covers both — ~20 lines total.

3. **`typeIsConvertibleTo` is not enough for a new RecTy.** `ClassRecTy` also
   needs `typeIsA`, or `TypedInit::getCastTo` asserts when widening
   `class<FmtA>` to `class<Fmt>`. Not mentioned anywhere on this page; found by
   crashing.

4. **A class name in value position must be the *last* resort.** A def and a
   class may share a name. Resolving the class before the def self-reference
   path silently changes what existing `.td` means. Ordered after, the only
   behavior change is on the path that previously ended in
   `Variable not defined` — strictly additive. This is the subtlest thing in the
   patch and the page did not anticipate it.

5. **The primary-template trick works.** `isSubClassOf("W")` holds on every
   instantiation, and the mangled `W$0` names never reached any of the three
   name-matching backend sites from §3 — the hash gate would have caught it.

### Still open

- Stage 5 as scoped: no perf measurement on X86/AArch64, no `ProgRef.rst`, no RFC.
- The fourth and fifth positions (variadic argument lists; parameter types that
  depend on earlier parameters) remain unimplemented, by decision.
- `getSuperClasses()` name leakage into `SearchableTableEmitter`'s arity check
  is unhandled — it bites only a templated `SearchableTable` class.

## Risks

- **"Just use a multiclass."** The likeliest upstream response, and — as the
  `Combine.td` experiment proved — correct far more often than the idea list
  assumed. The answer must be a case where the *base class* varies, hence
  SystemZ. Have the line-count diff and the identical `.inc` hashes ready
  before arguing, not after.
- **Eager parse-time resolution is load-bearing.** TableGen resolves inheritance
  as it parses; deferring it touches the oldest invariant in the parser. Stage 3
  is where this either works or the feature shrinks to value-position-only.
- **Type checking gets weaker.** Anything reached through an unbounded class
  parameter is unchecked until instantiation, and errors surface at a
  confusing location. Mitigation is to require bounds; the cost is that the
  most flexible form is the least safe.
- **`deftype` already said no** (`TGParser.cpp:4098`) — see *Still to check*.
- **Position 1 is the widest change and the one the demo needs.** It alters a
  record's field set after parse time. Positions 2 and 3 are narrower. Staging
  puts 1 first for payoff reasons, which means the riskiest work comes first —
  accept that deliberately, and keep stage 4 (30 → 11) as a shippable stopping
  point in case stage 5+ stalls.

- **Build times.** TableGen runs on every LLVM build. Deferred instantiation
  adds resolution passes; measure `.inc` generation time for the biggest
  targets (X86 89k lines of `.td`, AArch64 119k) and keep the regression
  visible.

## Relation to the other ideas

- **Idea 2 (namespaces)** and **idea 6 (delegation/composition)** are the same
  family: all three are about abstracting over declarations rather than over
  values. Whatever comes out of §4 constrains both.
- **Idea 7** (`let mayStore = 1 in defm : ...`) is adjacent — it also wants to
  reach into a record's field set from outside. This fork already carries
  `LetMode::{Append,Prepend}` (`TGParser.h:34`), so that area is live.
- **[MIR combines in td](../mir-combines-in-td/index.md)** notes its rule
  families duplicate the same way. If both land, the higher-order rule template
  applies to both; pick one as primary.
