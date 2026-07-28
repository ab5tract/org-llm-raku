# Extracting `org/` into the `org-llm-raku` submodule

**Landing point:** raku-intellij-plugin commits `77a37c1a` (remove `org/`),
`8d1f56b4` (mount the submodule), `e5605a95` (re-point references) — and the new
repository [`ab5tract/org-llm-raku`](https://github.com/ab5tract/org-llm-raku),
public, `main`.

## What changed

`org/` is no longer a directory in this repo. It is a gitlink to `org-llm-raku`, whose
contents sit one level deeper than before:

```
before   org/llm/{traces,research,report}/
after    org/llm/raku/{traces,research,report}/
```

Two motivations, and they are different from each other. Raku developers who would
rather not have AI-authored artifacts in their checkout can now clone the plugin and
leave the submodule uninitialised — the build does not read this tree, so that costs
them nothing but the contents. And the parts that are about *Raku* rather than about
this plugin, above all the named-argument work, are consumable by any Raku project
without dragging an IntelliJ plugin along.

The `raku/` segment is why the split is worth doing once rather than twice: the path
now reads *knowledge of kind `llm`, about language `raku`*, so anything learned about
a second language gets a sibling instead of being stirred in.

## How, and why that way

`git subtree split -P org` rather than a fresh import. It rewrites the 15 commits that
touched `org/` into a root-level history, keeping messages, authors and dates. The
`llm/` → `llm/raku/` move is a single commit on top, so the pre-move paths stay
reachable — which matters, because the frozen experiment data still names them.

A side effect worth recognising rather than fixing: several commit messages in the new
repo describe plugin work that is not in the new repo (`Convert syntax highlighting to
Kotlin…` carries only the trace file it also touched). That is what splitting a mixed
commit looks like. Rewriting the messages would trade honest provenance for tidiness.

## The two things that actually broke

Neither was visible in a diff. Both are the same shape: **a path assumption that the
extraction invalidated silently.**

**1. `40-measure-paired.raku` counted `.parent` hops.**

```raku
my $repo = $*PROGRAM.parent.parent.parent.parent.parent;   # before
my $repo = Corpus::repo-root($*PROGRAM.IO);                # after
```

Five hops reached the repo root from `org/llm/research/raku-tokens/`. From
`org/llm/raku/research/raku-tokens/` it needs six, so the script would have run the
paired arms with a `cwd` one level too high, every arm would have found nothing, and
both arms would have agreed on empty output — *the agreement check would have passed*.
`lib/Corpus.rakumod`'s own comment warns against counting `..` segments; the warning
was there and one script had not taken it.

The check that this is now right: re-running produced `92-paired.tsv` and
`93-paired-rollup.txt` **byte-identical** to what was committed before the move.

**2. `Corpus::repo-root` had nothing to walk up to.**

The measured corpora are the *plugin's* files — `testData/`, `scripts/`,
`src/main/java/`, `docs/`. This tree contributes only `traces/`. Mounted at `org/` the
upward search for `settings.gradle.kts` still finds them; cloned standalone from
GitHub it cannot, and the old failure said `could not locate repo root`, which tells a
newcomer nothing. It now takes `CORPUS_REPO_ROOT` (matching the existing
`CORPUS_PYTHON_STDLIB` / `CORPUS_RAKU_ECOSYSTEM` overrides) and its failure explains
the submodule relationship.

## What was left alone, and why

`level3/runs/**` and `90-corpus-per-file.tsv` are **measured record**, not source.
Rewriting `org/llm/traces` to `org/llm/raku/traces` inside a model's `attempt-01.raku`
would make `96-level3.tsv` a tally of code that was never run. They keep the paths they
were written with; `level3/README.md` and `99-notes.md` say so.

The visible cost is that `60-verify-corpus.raku` now reports `prose-markdown` at 26%
on the host that produced the data. That is the verifier working: it looks files up by
recorded path, and 14 of them moved. Re-resolved at the new path, 13 of the 14 are
byte-identical and the fourteenth is `traces/README.md`, which this extraction edited.
The point of recording a digest per file is that this distinction is *checkable*
rather than assumed — so check it before concluding a corpus rotted.

One exception was made deliberately: `tasks/l5-trace-words/SPEC.md` **was** rewritten,
so the task set stays runnable. `level3/README.md` names it as the one place its
"the spec an arm was given, and nothing else" claim no longer holds literally.

## Working across two repos now

The failure mode to watch for is ordinary and easy: **an edit under `org/` commits to
`org-llm-raku`, and this repo needs a follow-up commit to move the gitlink.** Commit
and push the submodule first, or the gitlink names an object nobody else can fetch.

```bash
git submodule update --init          # org/ is empty: clone lacked --recurse-submodules
git -C org status                    # where your org/ edits actually live
```

`.gitmodules` records the SSH URL, matching `origin`. Anonymous HTTPS clones of the
submodule work regardless; only pushing needs the key.

Related: `raku-named-args-corpus.md` (the general-Raku work that motivated wanting
this tree consumable elsewhere), `org/llm/raku/research/raku-tokens/99-notes.md`
(what the extraction did to the recorded data, item by item).
