# ml-model-iteration — an Agent Skill for disciplined ML model iteration

A [Claude Agent Skill](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
that makes Claude improve a predictive model one verifiable step at a time: champion and
challenger iterations, a data-driven evaluation protocol, and a diagnosis before every new
feature. It was distilled from a forecasting project (daily demand per entity, about one year
of history, demand strongly tied to temperature and the seasonal profile), but it is written to
be independent of domain, tools and infrastructure.

## What it does

When the skill triggers, Claude first prepares the project once (steps 0–5), then runs the
iteration loop (steps 6–9) for every challenger, and lets the user decide every promotion:

| Step | What happens |
|---|---|
| 0. Use case | Which decision the predictions feed, what errors cost, required model properties (e.g. a smooth, strictly monotonic price response for price optimisation) — written down as acceptance checks; how the decision is made today, with its documentation |
| 1. Data, stack, infrastructure | Short interview (local files and notebooks, Databricks and Delta tables, warehouse, …); available resources and their cost structure, and how the user weighs their own time against compute or cloud costs; critical feedback on the choices, above all on the iteration cycle length |
| 2. Data exploration | Potential (what is explainable beyond simple rules) and main challenges (cold starts, anomalies, short history, drivers unavailable at decision time, dependence) |
| 3. Evaluation design | Information cutoff from the use case; validation split chosen from the data (random, grouped, time-based, purged, spatial, distance matching, nested); disjoint selection and verification partitions |
| 4. Non-ML baseline | Today's method reproduced from its documentation, plus simple rule-based benchmarks — the reference for everything after |
| 5. Iteration 0 | The smallest model that meets the requirements; its run time is the cycle length; it is the first champion |
| **6–9. Iteration loop (repeated)** | One change per candidate with a stated hypothesis; per round a single challenger, parallel challengers or a bounded search (feature selection, tuning), chosen by expected information gain against run time and cost; parallel winners re-tested one on top of the other; acceptance checks as gates; promotion only if better in the selection partition and confirmed in the verification partition, with a block-bootstrap significance test; diagnosis before the next feature; the user promotes or rejects |

## When it triggers

Requests such as improving or tuning a model, adding or dropping features, comparing a new
model version with the current one or a baseline, asking whether a change is "really better",
setting up a backtest, cross-validation or train/validation/test split, translating a business
use case into model requirements, or diagnosing why a model loses to a simple benchmark.

## Repository layout

```
SKILL.md                              the workflow, decision rules and reporting (loaded when the skill triggers)
references/
  evaluation-protocol.md              information cutoff, benchmarks, rolling origin, partitions,
                                      block bootstrap (language-neutral algorithm), metrics,
                                      canonical methods and deviations
  validation-split-design.md          choosing the split from the data: deployment target,
                                      dependence diagnostics, method table, acceptance checks
  data-exploration.md                 what to explore and what the exploration must hand over
  diagnosis-and-pitfalls.md           how to diagnose a gap, worked examples, leakage and
                                      engineering pitfalls
```

`SKILL.md` stays short enough to load fully; the references are read only when a step needs
them.

## Installation

Claude Code looks for skills in `~/.claude/skills/<skill-name>/` (personal, all projects) and
in `<project>/.claude/skills/<skill-name>/` (one project). To use this repository as the
single source, link it instead of copying it, so edits here take effect immediately:

```bash
ln -s /Users/stm/code/ML_workflow_skill ~/.claude/skills/ml-model-iteration
```

If a copy already exists at that path, remove it first.

Skills do not sync across Claude surfaces. For claude.ai, upload the folder as a zip file
(without `.git`) under Settings > Features; for the Claude API, upload it through the Skills
API. See [Agent Skills — where Skills work](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#where-skills-work).

Note for edits: the `description` in the `SKILL.md` frontmatter may be at most 1024
characters (currently 987) and decides when the skill triggers.

## Design principles

- **No assumptions about tools or infrastructure.** The skill asks how the user works and
  adapts; algorithms are specified in plain language, not as code for one stack.
- **The user decides.** Claude presents evidence and a recommendation; it does not promote
  models, commit or publish on its own.
- **Established methods, named and cited.** Rolling-origin evaluation (Tashman 2000), block
  bootstrap (Carlstein 1986, Künsch 1989), structured cross-validation (Roberts et al. 2017),
  validity of k-fold for autoregressive models (Bergmeir et al. 2018), hv-block CV (Racine
  2000), purged CV (López de Prado 2018), distance-matching CV (Milà et al. 2022, Linnenbrink
  et al. 2024), nested CV (Varma & Simon 2006). Where the skill deviates, it says so.

## Status

Drafted from one project and not yet tested on independent tasks. Feedback from real use —
especially on other domains and stacks — is the next step.
