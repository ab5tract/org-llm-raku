# org-llm-raku

Agent-authored, durable knowledge about **Raku** — investigation traces, measured
experiments, and the findings they produced. Written by AI agents working on
[raku-intellij-plugin](https://github.com/ab5tract/raku-intellij-plugin), for whoever
comes next, human or otherwise.

It lives in its own repository for two reasons. Raku developers who would rather not
have LLM artifacts in their checkout can clone the plugin and skip the submodule
entirely. And the parts that are about *Raku* rather than about that plugin — most of
all the named-argument work — are useful to any Raku project, without dragging an
IntelliJ plugin along with them.

## Layout

```
llm/raku/traces/     investigation traces: root causes, dead ends, environment knowledge
llm/raku/research/   experiment harnesses, raw data, working notes — method and mess
llm/raku/report/     the tidied findings those experiments produced — read for answers
```

The `llm/raku/` prefix is deliberate. This is designed to be mounted at `org/` in a
consuming repo, so paths read `org/llm/raku/traces/…`: knowledge of kind *llm*, about
language *raku*. Anything learned about another language gets a sibling rather than
being stirred in with this.

Start at [`llm/raku/traces/README.md`](llm/raku/traces/README.md), which gives a
reading order.

## Using it

```bash
git submodule add git@github.com:ab5tract/org-llm-raku.git org
```

Existing clones of a repo that already has it:

```bash
git submodule update --init          # after a plain clone
git clone --recurse-submodules …     # or ask for it up front
```

Skipping it costs you nothing but the contents. Nothing in raku-intellij-plugin's
build depends on this tree being present.

## The one thing to take away

**Raku silently swallows named arguments it does not understand.** Methods carry an
implicit `*%_`, so `.dir(:recursive)`, `.dir(:r)`, `.pick(:seed)` are all accepted, all
ignored, and all return confident, well-formed, wrong output. Measured over 48 blind
agent arms, this was 3 of the 4 Raku first-attempt failures, against 0 failures for
Python — and three of the four exited 0 while printing plausible numbers.

The full measurement is in [`llm/raku/report/raku-tokens/`](llm/raku/report/raku-tokens/README.md);
the practical response — how to ask the running Rakudo what a method actually declares —
is in [`llm/raku/traces/raku-named-args-corpus.md`](llm/raku/traces/raku-named-args-corpus.md).

## Running the research harness

`llm/raku/research/raku-tokens/` measures corpora that mostly belong to the *plugin*
repo (`testData/`, `scripts/`, `src/main/java/`, `docs/`). Checked out as the plugin's
`org/` submodule it finds them by walking upwards. Cloned on its own it cannot, so say
where a checkout is:

```bash
CORPUS_REPO_ROOT=/path/to/raku-intellij-plugin raku llm/raku/research/raku-tokens/20-measure-corpus.raku
```

The tokenizer vocabularies are not committed (`10-fetch-vocab.raku` downloads them).
Recorded results are portable by construction — every path is relative to a run-time
resolved root, with a content digest beside it, and `60-verify-corpus.raku` reports how
much of the recorded corpus is present and unchanged on your machine rather than
pretending it matches.

## Provenance

Split out of raku-intellij-plugin's `org/` directory with `git subtree split`, so the
commits here are the original ones, with their original messages and dates. A handful
of those messages describe plugin work that is not in this repository — that is what a
subtree split of a mixed commit looks like, and it is left alone rather than
rewritten. The rearrangement from `llm/` to `llm/raku/` is a single commit on top.
