# Judging a claim against its evidence

`L4.CLAIM_IS_SUPPORTED_BY_ITS_EVIDENCE` ships with `run: null` and no default
judge. That is a deliberate refusal, not an unfinished edge, and this note is
the half that keeps the refusal honest: a rule whose only documentation is "you
supply the oracle" hands every adopter the same unsolved problem, and most of
them will wire in something that can only ever pass.

## Why the catalog names no judge

The catalog names no generator either, and for the same reason. A judgment that
costs money or needs a credential belongs to the repository running it: the key
is theirs, the bill is theirs, and the verdict has to be reviewable in their
diff rather than produced inside a binary they installed from a tag. A rule
shipping a default judge would also decide the question's wording for everyone,
and the wording is the check.

What the rule adds over a plain CI step is everything around the command: the
reason printed where it fails, a mutation proving it still fails when it
should, a ratchet for the debt already there, and a policy that cannot be
quietly loosened.

## The reference oracle: a TypeSafe Jev gate

The judge this rule was written against is a [TypeSafe](https://typesafe.ai)
Jev judgment — a calibrated typed answer to one narrow question, with a
confidence number attached. The wiring below is the shape the rule expects from
any oracle, whether or not it is Jev.

```yaml
# .software-factory/policy.yaml
  L4.CLAIM_IS_SUPPORTED_BY_ITS_EVIDENCE:
    enabled: true
    options:
      run: node tools/jev-gate.mjs --request gates/jev/claims.json --expect relation=supports --min-confidence 0.8 --cache gates/jev/cache.json
```

The request is a file in the repository, not a string built at runtime, so the
question and what counts as passing it are both reviewable:

```json
{
  "state": { "claim": "...", "evidence": "..." },
  "questions": {
    "relation": {
      "type": "choice",
      "instructions": "Does the evidence support the claim?",
      "criteria": {
        "supports": "the evidence contains the effect the claim promises",
        "contradicts": "the evidence shows the opposite",
        "says_nothing": "the evidence is silent about the claim"
      }
    }
  }
}
```

Four properties make that command a check rather than a rubber stamp. Each one
is the answer to a way this rule fails quietly.

**The deterministic part runs first.** Locating the claim marker, resolving the
gate it names and loading that gate's report are string operations. They belong
in the command's own code, exiting nonzero on their own terms. The judge sees
only the residue nothing else can decide: whether this body of evidence
supports this sentence. A judgment placed where a `grep` belongs costs money and
can flake.

**A confidence floor, and a low verdict is a finding.** `0.8` is the starting
point. The failure line has to distinguish "landed on the wrong choice" from
"landed on the right one without conviction", because the tuning evidence is
the difference between them. Lowering the floor to go green is
`L2.POLICY_ONLY_TIGHTENS`' problem, and it will notice.

**The verdicts are cached and committed.** An uncached request calls the API,
which makes the check nondeterministic, network-dependent and priced per run —
none of which a gate can be. Hash the model, state and questions, commit the
cache next to the request, and the check replays exactly and for free. A
changed question hashes to a new key and calls out again, which is the correct
behaviour: a rewritten question is a different check.

**The oracle is proven on both sides.** An oracle seen only passing is
unvalidated; it may be approving anything. Keep a second request alongside the
real one carrying the *negated* criterion, and require the real one to pass and
the planted one to fail. That is the same argument as `sf verify` making a
mutation fixture mandatory, applied to the one check `sf verify` cannot reason
about from a query — which is why the mutation fixture for this rule points its
`run` at a command that fails, and why proving the rule needs
`sf verify --allow-commands`.

No secrets go into the state. The judge reads what it is handed and the handoff
is a committed file.

## Why it is switched off in this repository

This repository has not written that oracle yet. Enabling the rule with nothing
in `run` would be a rule lying about its own coverage, which is precisely what
`L5.NO_INERT_RULE` exists to refuse, so the decision is written down in
[`docs/rules.md`](../rules.md) instead and the rule stays off until the judge
exists. The marked claims themselves are already policed one rule down:
`L4.CLAIM_CITES_ITS_EVIDENCE` requires every promise to name a declared gate,
and `L3.GATE_HAS_FRESH_EVIDENCE` expires that join when the implementation
moves. What neither of them can do is read the sentence.
