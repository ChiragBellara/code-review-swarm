# Code Review Swarm — Scoping Notes

Status: pre-build, pressure-testing the idea. Nothing built yet.

## The idea

A multi-agent system where several specialized agents (security, performance,
style/convention, test-coverage) independently review the same code diff in
parallel, and a synthesizer/judge agent merges their findings, dedupes
overlaps, ranks by severity, and explicitly surfaces disagreements between
agents rather than silently resolving them.

## Why this project (portfolio framing)

The trap with multi-agent portfolio projects is building something where
multiple agents don't actually earn their keep over a single agent with a
checklist. This project is defensible because:
- The specialist lenses are naturally independent and parallel (mirrors how a
  human review team works).
- "Agents disagree, here's the conflict" is genuinely interesting to demo —
  most commercial review tools hide this rather than showing it.

## Architecture (not committed)

- Built on the Claude Agent SDK, orchestrator-worker pattern: specialists fan
  out in parallel over the diff, synthesizer fans back in.
- Each specialist returns structured output (file, line, severity, category,
  claim, confidence) rather than free text, so synthesis is tractable.
- Context design tradeoff: diff-only vs. diff+touched-files vs. an on-demand
  "fetch more context" tool. Leaning toward diff+touched-files by default,
  with on-demand fetch as a later addition.
- Synthesizer "annotates conflicts, not silently resolves them" when two
  agents disagree (e.g. performance agent wants a cache, security agent flags
  that cache as a data-leak risk) — resolving genuine tradeoffs by committee
  overclaims what the system can actually judge.

## Validation plan

- Primary: inject known bug categories (SQL injection, off-by-one, missing
  null check, N+1 query, ~5 categories total) into clean real PRs, run the
  swarm against clean vs. bugged versions, measure precision/recall.
- Secondary: test against real merged PRs that needed a follow-up bugfix —
  would the swarm have caught the original bug?
- Report false-positive rate as prominently as catch rate.

## Proposed stack

Python backend/pipeline, Claude Agent SDK for orchestration, FastAPI to serve
it, React frontend (matches existing portfolio stack) for the eventual demo
dashboard.

## Phased plan

0. Single-agent spike to validate structured output works reliably.
1. Core parallel swarm + synthesizer as a CLI tool.
2. Validation/benchmarking with the bug-injection test set.
3. Web demo wrapping the validated pipeline.
4. Polish: on-demand context tool, cost/latency benchmarking of parallel vs.
   sequential agents, configurable reviewer panel.

Instinct: get phases 0–2 fully solid (validated CLI, real precision/recall
numbers) before touching the web demo — a working tool with honest metrics is
more credible than a shiny UI around something unvalidated.

## Pressure-test findings (first pass)

**Architecture — mostly sound, one real risk.**
Fan-out/fan-in fits independent lenses well. The hard unsolved part: deduping
overlapping findings across agents when they use different line numbers,
wording, or framing for the same underlying bug. Exact-match dedup won't
work — likely needs semantic matching (file + overlapping line range + same
root cause) in the synthesizer, and that logic needs its own testing, since
over-merging defeats the "surface disagreement" pitch. Also: confidence
scores from different specialist prompts aren't inherently comparable (a
security agent's 0.8 ≠ a style agent's 0.8) — don't rank by raw confidence
without normalizing.

**Validation — good instinct, missing the one comparison that justifies the
project.**
Bug-injection + precision/recall + false-positive rate is the right shape.
Two gaps:
1. Missing a **single-agent-with-a-checklist baseline** run on the same
   injected bugs. The thesis is "multi-agent earns its keep over one agent
   with a checklist" — current plan proves the swarm catches bugs, not that
   it beats the simpler alternative. This comparison is the actual credible
   claim.
2. Leakage risk: if the injected-bug categories and the specialist prompts
   are designed in the same sitting, it's grading its own homework. Write
   injections blind to the specialist prompts (or vice versa), or have
   someone else pick/write a chunk of the injected bugs.
3. Sample size: 5–10 injected bugs per category is noise. Aim for 30+
   instances per category for numbers worth citing.
4. The "real merged PRs with follow-up fixes" secondary test has selection
   bias (these are bugs that already escaped review) — fine as a secondary
   signal, don't lean on it for the headline number.

**Scope cuts for solo build.**
Phasing (0–4) is right. Additionally cut the "configurable reviewer panel"
entirely (low payoff, adds surface area). Keep cost/latency benchmarking of
parallel vs. sequential — cheap to collect, directly supports the "do
multiple agents earn their keep" argument quantitatively (cost/time overhead
vs. detection lift).

**What to lead with in the eventual writeup.**
Not the disagreement-surfacing feature alone. Lead with the single-agent
baseline comparison, with one concrete disagreement example as the
illustrating artifact: "here's the precision/recall delta vs. a single
model-with-a-checklist, and here's a case where two specialists disagreed and
why resolving it by committee would've been wrong" is a stronger portfolio
claim than "agents disagree, neat."

## Open next steps

- Design the bug-injection category set (and who writes it, to avoid
  leakage).
- Design the single-agent baseline prompt for comparison.
- Decide on the diff+touched-files context strategy concretely (what counts
  as "touched," size limits).
