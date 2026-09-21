# Judging a claim against its evidence

Some claims a repository makes about itself are not decidable by a query.

`L4.CLAIM_CITES_ITS_EVIDENCE` requires every marked promise to name a declared
gate, and `L3.GATE_HAS_FRESH_EVIDENCE` expires that join the moment the
implementation moves. Neither of them reads the sentence. A report can say
`passed` without ever containing the effect the sentence promises, and a
sentence can be quietly strengthened — "handles retries" becomes "handles
retries under partition" — with no digest changing anywhere.

`L4.CLAIM_IS_SUPPORTED_BY_ITS_EVIDENCE` is the rule that closes that gap, and
it is the only rule in the catalog that ships with no check of its own. It runs
a command the repository owns:

```yaml
  L4.CLAIM_IS_SUPPORTED_BY_ITS_EVIDENCE:
    enabled: true
    options:
      run: node tools/claim-judge.mjs
```

This page is how to write that command, because a rule whose entire
documentation is "you supply the oracle" hands every adopter the same unsolved
problem, and the cheapest thing to build is an oracle that can only pass.

## Why the catalog names no judge

The catalog names no code generator either, for the same reason. A judgment
that costs money or needs a credential belongs to the repository running it:
the key is theirs, the bill is theirs, and the verdict has to be reviewable in
their diff rather than produced inside a binary they installed from a tag.
Shipping a default would also fix the wording of the question for everyone —
and with this kind of check, the wording *is* the check.

What the rule adds over writing the same command as a plain CI step is
everything around it: the reason printed where it fails, a mutation proving it
still fails when it should, a ratchet for the debt already there, and a policy
that cannot be quietly loosened to make it stop.

## The reference oracle: a TypeSafe Jev judgment

[Jev](https://typesafe.ai) is TypeSafe's judgment model: you hand it a state
and a typed question, it returns one of your named choices plus a calibrated
confidence. That shape is what makes it usable as a gate — a free-text LLM
answer has to be parsed and believed, a typed choice with a probability can be
compared against a threshold by four lines of shell.

The rest of this page uses Jev concretely. Any oracle with the same contract
works; the four properties below are what separate a check from a rubber stamp,
whichever model answers.

### 1. Configure the credential

```sh
export TYPESAFE_API_KEY=...        # never committed; in CI, a secret
export TYPESAFE_BASE_URL=https://api.typesafe.ai   # optional, this is the default
```

The command must exit nonzero when the key is missing rather than skipping the
check. A judge that silently passes when it cannot run is worse than no judge:
it reports green for the one class of defect nobody else is looking at. That is
also why the rule's `fix` says an oracle nobody can run *is itself the
finding*.

### 2. Write the question as a committed file

Not a string built at runtime. The question and what counts as passing it are
both reviewable in the diff, and the cache key below depends on them:

```json
{
  "state": {
    "claim": "Adopting the factory in an existing repository ends green.",
    "evidence": "…the report for the `adoption` gate, verbatim…"
  },
  "questions": {
    "relation": {
      "type": "choice",
      "instructions": "Does the evidence support the claim? Judge only what the evidence shows.",
      "criteria": {
        "supports": "the evidence contains the effect the claim promises",
        "contradicts": "the evidence shows the opposite of the claim",
        "says_nothing": "the evidence is silent about what the claim promises"
      }
    }
  },
  "expect": { "relation": "supports" },
  "minConfidence": 0.8,
  "cache": "cache.json"
}
```

One narrow question. Criteria that genuinely separate the options — if two
choices can both be argued for the same input, the verdict is noise with a
confidence number attached. State goes in named fields the instructions point
at, and nothing secret goes in it: everything in `state` leaves the machine.

### 3. Wire it into the policy

```yaml
# .software-factory/policy.yaml
  L4.CLAIM_IS_SUPPORTED_BY_ITS_EVIDENCE:
    enabled: true
    options:
      run: >-
        node tools/claim-judge.mjs
        --request gates/jev/claims.json
        --expect relation=supports
        --min-confidence 0.8
        --cache gates/jev/cache.json
```

Then `sf lock`, because the policy hash covers it. Command rules are refused by
default and need `sf check --allow-commands`, which is the deliberate act of
saying this repository's checks may execute something.

Keeping `--expect` and `--min-confidence` on the command line rather than only
in the request file costs a line and buys a lot: the policy diff shows what the
check accepts, so loosening the threshold is a change to a locked file instead
of a change to a JSON blob nobody re-reads.

### 4. The command's contract

| | |
| :-- | :-- |
| exit `0` | every expected question landed on an expected choice at or above the floor |
| exit `1` | wrong choice, or right choice below the confidence floor — the finding |
| exit `2` | usage, request, cache or API failure — also a finding, not a skip |

Print the verdict, the choice and the confidence on the passing line too. `sf`
surfaces the command's output in the finding, and "which claim, judged how
confidently" is the whole diagnostic.

## The four properties that make it a check

**The deterministic part runs first.** Finding the claim markers, resolving the
gate each one names, and loading that gate's report are string operations.
They belong in the command's own code, failing on their own terms with their
own message. The judge sees only the residue nothing else can decide: whether
*this* evidence supports *this* sentence. A judgment placed where a `grep`
belongs costs money, can flake, and teaches people to distrust the rule.

**A confidence floor, and a low verdict is a finding.** `0.8` is a reasonable
starting floor. The failure line must distinguish "landed on the wrong choice"
from "landed on the right one without conviction", because those call for
different fixes — the first is a bad claim, the second is usually a badly
posed question or thin evidence. Lowering the floor to go green is a change
`L2.POLICY_ONLY_TIGHTENS` will notice and refuse.

**Verdicts are cached and committed.** An uncached request calls the API, which
makes the check nondeterministic, network-dependent and priced per run — none
of which a gate can be. Hash the model, state and questions, commit the cache
next to the request, and every replay is exact and free. A changed question
hashes to a new key and calls out again, which is the correct behaviour: a
rewritten question is a different check, and its old verdict was about
something else.

**The oracle is proven on both sides.** An oracle only ever seen passing is
unvalidated — it may be approving anything, and nobody would know. Keep a twin
request carrying the **negated** criterion and require the real one to pass and
the twin to fail. Wrong on either side is a broken judge. This is the same
argument that makes a mutation fixture mandatory for every other rule, applied
to the one check `sf verify` cannot reason about from a query. It is why the
mutation fixture for this rule points `run` at a command that fails, and why
proving it needs `sf verify --allow-commands`.

## What it cannot do

A calibrated judgment is not a proof. Confidence summarises the probability
distribution over *this* question, not the correctness of the work, so a
consequential low-confidence verdict belongs in front of a person rather than
under a lowered floor. And the judge only knows what the evidence file says: it
cannot tell you the test that produced it was meaningful. That is `sf verify`'s
job, one layer down.

## Why this repository has it switched off

No judge has been written here yet. Enabling the rule with nothing in `run`
would be a rule lying about its own coverage, which is exactly what
`L5.NO_INERT_RULE` exists to refuse — so the decision is written down in
[`docs/rules.md`](../rules.md) and the rule stays off until the oracle exists.
The marked claims themselves are already policed by the two rules this page
opened with. What neither of them can do is read the sentence.
