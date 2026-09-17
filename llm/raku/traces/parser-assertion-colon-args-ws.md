# Regex assertion colon-args: whitespace before '>' swallowed the file

**Fix:** commit `d904d25c` on `richard-fixer`. Regression test:
`testData/parsing/regex-assertion-arg-ws/` + `RegexAssertionArgWsTest`.

**Prerequisite reading:** `parser-generated-lexer-architecture.md`.

## Symptom

Rakudo's `Raku/Grammar.nqp` `statement-control:sym<loop>` token broke parsing
with no recovery. Token dump showed the multi-line assertion's closing `>`
lexed as `STRING_LITERAL_QUOTE_CLOSE`, then one `BAD_CHARACTER` to EOF.

## Root cause

Minimization (constructs tested one at a time: `:s''`, curly-quote literals,
`:my int $x;`, interpolated args, nested alternations — all fine) isolated:
**any whitespace between an assertion's colon-argument list and the closing
`>`** — `<.malformed: "x"   >` or the multi-line form. `raku -c` accepts
these. The grammar's `rxarglist` ended without consuming trailing ws, the
wrapper's `'>'` never matched, and error recovery collapsed everything.
The paren form `<.m("x" )>` was unaffected.

## Fix shape — BOTH machines this time

Adding trailing ws to the lexer alone (`MAINBraid._254_rxarglist` calling
rule 18/ws after arglist) produced "Tokens were not inserted into the tree":
the token-stream SHAPE changed, so the separate parser machine
(`RakuParser.rxarglist_157`) also needed a matching hand-edit (`ws_258`
call). Grammar mirror: `token rxarglist` gains `<.ws>` after `<.arglist>`
in `tools/p6-grammar-to-idea/perl6.pm6`.

## Transferable principles

1. A lexer-machine hand-edit that changes the emitted token sequence (not
   just token boundaries/types in-place) usually needs a matching
   parser-machine hand-edit — they are generated from the same grammar and
   communicate by exact token-shape agreement (see also
   `parser-nqp-const-term.md`, where a zero-width NO_ARGS emission kept the
   shape compatible instead).
2. Parse-output changes require STUB_VERSION + words-scanner bumps
   (`ParserChangeVersionGuardTest` now enforces the pairing). Gotcha found
   here: the guard must read version constants from SOURCE — const inlining
   plus Gradle compile avoidance served stale compiled values in practice.
