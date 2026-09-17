# nqp::const:: names mis-lexed as listops (swallowed `&&`)

**Fix:** commit `0161a126` on `richard-fixer`. Regression test:
`testData/parsing/nqp-const-term/` + `NqpConstTermTest`.

**Prerequisite reading:** `parser-generated-lexer-architecture.md`.

## Symptom

User report: highlighting (and PSI) break at the second `&` in `&&` in
Rakudo-style code:

```raku
if nqp::objprimspec($rev) == nqp::const::BIND_VAL_STR
  && $rev ~~ /^ v? $<vnum>=\d+ $<plus>='+'? $/ -> $m { ... }
```

Token dump showed `CONTEXTUALIZER('&')` + `VARIABLE('&')` followed by one
giant `BAD_CHARACTER` to the end of the statement.

## Root cause

`nqp::const::BIND_VAL_STR` went through the generic `term_name` listop path
(`_105_term_name`), which tries `<args>`. A listop's argument list may
legitimately begin with a bare-sigil term (`&` alone is a valid anonymous
variable term — real Rakudo rejects `foo && 42` and even `nqp::time && 42`
with "Two terms in a row" for exactly this reason). So the mis-parse is
faithful for ordinary names; the bug is that Rakudo's grammar has a dedicated
**`term:sym<nqp::const>`** (`'nqp::const::' <identifier>`) that takes **no
argument list**, and the plugin grammar lacked it. Empirical ground truth:
`nqp::const::FOO && 42` compiles; `foo && 42` and `nqp::time && 42` do not.

## Fix shape

`MAINBraid.java` `_105_term_name()` case 10 (the `<args>` step): when the
just-lexed `SUB_CALL_NAME` starts with `nqp::const::`, emit the zero-width
`NO_ARGS` structural token (the parser requires it to reduce the call term —
skipping args entirely produced "Tokens were not inserted into the tree")
and jump to the success state, bypassing rule 117. Helper
`startsWithNqpConst` sits right below the rule.

**Grammar mirror NOT applied:** the external `p6-grammar-to-idea` checkout
was absent; `term:sym<nqp::const>` must be added to `perl6.pm6` before any
regeneration. The commit message says the same.

## Transferable principle

The generic name/listop path is only correct for names that can actually be
listops. A qualified-name family with fixed arity (here `nqp::const::*`)
must bypass argument parsing, and the bypass must still emit the structural
tokens (`NO_ARGS`) the token-stream parser expects — the lexer and parser
are separate generated machines communicating through zero-width signals.

## Debug shortcuts that worked

Token-stream dump via bare `RakuHighlighterLexer` (no fixture), then
variant minimization (`foo && $b` / `foo() && $b` / `nqp::foo && $b` /
`nqp::const::X && $b` / `||`/`+`/`==` controls) — the newline in the user
report was a red herring; `raku -c -e` one-liners for compiler ground truth.
