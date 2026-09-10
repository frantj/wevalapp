# WPS benchmark — pre-run checklist

Forward-looking operational checklist for executing a WPS benchmark run (Run 6.x and
successors) from this repo. Last updated: 2026-09-10, after Run 6.8.

**Why this exists.** Every item below is already recorded somewhere — in
`build_leaderboard.py`'s docstring, in `docs/DECISIONS.md`, or in a per-run `STATUS.md`.
But it is recorded as *history* ("in Run 6.8, a 402 happened") or as *implementation
detail*, never as instructions to the next person. Two of these traps have now bitten
twice. This file is the checklist; the sources it points at remain authoritative.

**This file does not restate values.** Where a number or a rule lives in code or in the
decision log, this points at it instead of copying it — a copy has nothing enforcing it
and drifts silently. Per-run plan docs may restate freely; this one does not.

Repo note: the WPS material lives in `docs/jesper's docs/` in this fork. The decision log
(`docs/DECISIONS.md`), run plans and `STATUS.md` files live in the sibling
`Weval-WPS-Eval` and `microsoft-foundry-framerwork-workflow` repos.

---

## 1. Before committing budget

- [ ] **Get the run number right.** D17 defines the scheme: integer = criteria
      generation, decimal = same blueprint/criteria/judges with only model or agent
      config varying, letter suffix = re-execution of an existing config, and a study
      that changes the blueprint without advancing criteria gets **its own namespace and
      no run number**. Assign at folder-creation time, not after a collision. D17 exists
      because "Run 7" was informally claimed by two different things at once.
- [ ] **Confirm every SUT is reachable through the harness's existing path.** If a model
      is not on OpenRouter, this stops being a scoring run and becomes an infrastructure
      project — that is what blocked Run 6.7. Check before setting a date.
- [ ] **Confirm what stays constant.** Blueprint, criteria, judge panel *and judge
      versions*, `temp:0`, prompt order, scoring pipeline. Anything else moving means the
      result is not comparable to prior runs, and probably means it is not a decimal.
- [ ] **Pin temperature explicitly.** Every scored SUT to date is `temp:0`. Do not
      inherit a vendor example's sampling config. Some reasoning-tier deployments reject
      an explicit `temperature` outright — agent-side that is
      `WPS_MODEL_SUPPORTS_TEMPERATURE=0`.
- [ ] **Decide whether a drift control is needed.** D12 added one because inter-batch
      drift across Runs 6.1–6.3 was undetectable by construction — nothing had been
      scored twice. If this run lands well after the last one, re-scoring one prior SUT
      is the cheap guard.
- [ ] **State what question the run answers.** Runs 6.7 and 6.8 each have one. A run
      that only adds the newest models to the roster is legitimate maintenance, but label
      it as maintenance rather than framing it as a finding.

## 2. If an agent arm is in scope

- [ ] **`WPS_EMIT_TRACE=0`.** Run 6.1 logged four scenarios where internals leaked into
      responses and judge agreement went *negative*. Never set the trace flag for a
      scored run.
- [ ] **Re-run the attribution measurement if the base model changed.** Per D8/D19 the
      cross-country containment is prompt-layer behaviour, so a base-model swap is
      exactly what silently alters it. `measure_attribution.py` in the agent repo; D19
      records the acceptance bar (zero misattributed, zero unresolved).
- [ ] **Record provenance in the run manifest**, not just the model name: registry hash
      from `/wps/config`, commit SHA, endpoint, and auth mode. Per D9/D13 a run against
      localhost and one against the deployed URL are not interchangeable.

## 3. Running it

- [ ] **Raise the generation timeout up front.** Run 6.5's explicit advice after its
      repair pass: do not rely on a repair to absorb slow models, and budget for some
      models being materially slower than others in the same cohort.
- [ ] **Use `--skip-executive-summary`** if the summary is not being relied on for
      scoring. Run 6.8's `STATUS.md` records the exact invocation that worked, including
      `--eval-method llm-coverage`, `--gen-timeout-ms` and `--gen-retries`.
- [ ] **Keep the run sequential where concurrency is the known failure mode** — see the
      next section.

## 4. When it breaks

- [ ] **Expect a concurrency ceiling, not a timeout.** This has now happened twice:
      Run 6.5 (22 cells) and Run 6.8 (16 of 330, `402 in_flight_budget_exhausted` on a
      burst of concurrent judge calls, concentrated on two prompts). It is
      infrastructure, not model behaviour.
- [ ] **Repair the affected cells; do not re-run the whole thing.** `pnpm cli repair-run`
      — documented in the root `README.md` — regenerates and re-judges only the errored
      cells. Both prior runs recovered fully. Re-running all cells re-spends judge budget
      on cells that were already clean.
- [ ] **Know two `repair-run` side effects.** Caching is **off** by default for repairs
      (deliberate — fresh results). And it regenerates the executive summary, since it
      has no `--skip-executive-summary` flag; harmless if you were skipping the summary
      anyway, but not if you have one you want left alone.
- [ ] **Check whether every failure had the same cause.** Run 6.5's 22nd cell was an
      empty response, not a timeout — a raised timeout would not have prevented it.
- [ ] **Disclose a mixed-vintage result.** `repair-run` re-judges only what it
      regenerates, so cells can carry judgments from different sittings. Run 6.5 has two
      dates and says so; Run 6.8 recovered within minutes and says that instead. Either
      is fine; leaving it implicit is not.

## 5. Scoring and publishing

- [ ] **Do not compute the overall score by hand.** Use
      `Run 6/charts/build_leaderboard.py`. Its docstring is the authoritative
      methodology: the published figure is a **macro average across tiers** (mean of
      tier-means), not a pooled mean over cells, because the tiers are unequal in size.
      The docstring states plainly that pooling does *not* reproduce published values.
      Run 6.8 briefly published a figure from an ad hoc pooled mean and had to correct it
      the same day.
- [ ] **Run `build_leaderboard.py --check` before publishing.** It regenerates in memory,
      diffs against the current `leaderboard.json`, writes nothing, and exits non-zero on
      mismatch. It reproduces every previously published row, so a mismatch is a real
      finding about your new row.
- [ ] **Quote the adversarial criteria as T4-only.** WPS Guardrails and Substantive
      Pushback score near-ceiling on the non-adversarial prompts, so an all-prompts
      average hides the effect they exist to measure. `build_leaderboard.py` documents
      this and reports them T4-only.
- [ ] **Take tier membership from the blueprint's `# Tier:` comments, never from scenario
      ID numbers.** The numbers are misleading — the docstring's own example is that
      `scenario-16` is T2 despite its number.
- [ ] **Never fold a new bare SUT into `base_average` / `BASE_MODELS`.** Named as a
      specific misuse to avoid in D12, D16 and D20. Report against a separate cohort
      baseline instead.
- [ ] **`note` and `description` in `leaderboard.json` are hand-authored.** The build
      script carries them over by `published_id` and never invents them. A new row with
      empty note/description is a real gap to fill by hand, not a script bug.
- [ ] **Do not quote an ordering the margin cannot support.** Run 6.5's GLM-over-Qwen gap
      is 0.004 against a 3.6× response-length difference; its `STATUS.md` says not to
      quote it without a length-controlled check. Record per-SUT response lengths so this
      is checkable.
- [ ] **One bar per system on the leaderboard** (D12). A re-score is a drift tick on the
      existing bar, not a second bar.
- [ ] **Unscored maintenance does not go in the public version history** (D19). Every row
      there pairs a build with a score; a scoreless entry misreads as a new scored
      system. Record it in the decision log instead.

---

## Authoritative sources

| Topic | Source |
|---|---|
| Scoring methodology, tier macro average, T4-only, tier membership, note/description | `Run 6/charts/build_leaderboard.py` docstring |
| `repair-run`, `run-config` and all CLI flags | this repo's root `README.md` |
| Run numbering | `docs/DECISIONS.md` D17 |
| `base_average` misuse, one-bar-per-system, drift control | D12, D16, D20 |
| Agent attribution obligation on a base-model swap | D8, D14, D18, D19 |
| Provenance fields for a run manifest | D9, D13 |
| Per-run incidents, repairs and caveats | that run's `STATUS.md` |
