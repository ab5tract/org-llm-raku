# Indirect method names (`method ::($meth)`) swallowed the method body

**Fix:** commit `7791595a` on `richard-fixer`. Regression test:
`testData/parsing/indirect-method-name/` + `IndirectMethodNameTest`.

**Prerequisite reading:** `parser-generated-lexer-architecture.md`.

## Symptom

Rakudo's RakuAST action roles (`my role TermAction[$meth, $subname] { method
::($meth)($/) {...} }`) broke parsing with no recovery: the `{` after the
second paren group opened a `BAD_CHARACTER` run.

## Root cause

The DSL grammar's `morename` is simplified to `'::' <.identifier>?` — it
lacks Rakudo's indirect-name branch (`'::' '(' <EXPR> ')'`). So `method
::($meth)($/)` lexed the name as bare `::`, took `($meth)` as the signature,
and the genuine signature `($/)` derailed the routine rule. Ground-truth
nuance: `raku -c` rejects the form with "Name ::($m) is not compile-time
known" — a SEMANTIC error, i.e. the real grammar parses it; and the NQP
dialect (which the affected Rakudo sources are written in) accepts it
outright. An IDE should parse it.

## Fix shape — flat-token trick, ONE machine only

`MAINBraid._10_method_name` case 6: when the matched name ends with `::` and
`(` follows, consume the balanced paren group INSIDE the single ROUTINE_NAME
token (depth-only balancing, single-line, imbalance falls back to old
behavior). Because the token-stream shape is unchanged (still one
ROUTINE_NAME), the generated parser needed NO matching edit — contrast
`parser-assertion-colon-args-ws.md`, where emitting NEW tokens forced paired
edits in both machines. Tradeoff: `$meth` inside the name is not a separate
VARIABLE token (no resolve/highlighting inside the indirect name).

Grammar mirror: `token method_name` gains a gated alternative
`<.name>? <?before '::('> '::' '(' ~ ')' <.EXPR('i=')>` — on regeneration
this produces proper sub-tokens (both machines regenerate consistently),
superseding the flat-token compromise. First mirror draft was WRONG
(ungated `[ '(' ~ ')' ... ]?` would eat normal signatures as names) —
gate indirect-name mirrors on the `::(` prefix.

## Transferable principles

1. When extending the lexer machine, prefer shapes that keep the emitted
   token sequence compatible (grow an existing token, or emit the structural
   token the parser already expects) — that halves the hand-edit surface.
2. `raku -c` rejection is not proof of a syntax error: check whether the
   message is semantic ("not compile-time known") before concluding the IDE
   should also fail the parse. The NQP dialect is looser still.

## Follow-up (commit 29926f97): tokens/rules too, and inspection fallout

`token ::($meth_name)` went through `routine_name` (not `method_name`), which
lacked the fix — and failed DIFFERENTLY: no BAD_CHARACTER, just silently
wrong PSI (`ROUTINE_NAME('::')` + a bogus SIGNATURE holding `$meth_name` as
a shadowing parameter). Downstream that surfaced as an "Unused parameter"
inspection on the enclosing role's parameter. The hand-edit is now the
shared `indirectNameEnd()` helper used by both `_9_routine_name` and
`_10_method_name`; grammar mirrors on both rules. Lesson: a mis-parse that
produces no error nodes can still be wrong — inspection false positives are
a parse-bug smell worth tracing to PSI shape before touching the inspection.

Same commit also: regex-embedded `:my $x := ...;` side-effect declarations
exempted from unused-variable, and Rakudo-core detection got a file-level
path fallback (`CommaProjectUtil.isRakudoCoreFile`, matching `rakudo/src/`)
because the project-level flag misses Rakudo checkouts browsed from other
projects.
