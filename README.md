# ml-model-iteration — an Agent Skill for disciplined ML model iteration

A [Claude Agent Skill](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
that makes Claude improve a predictive model one verifiable step at a time: champion and
challenger iterations, a data-driven evaluation protocol, and a diagnosis before every new
feature. It was distilled from forecasting projects, but it is written to be independent of
domain, tools and infrastructure.

## What it does

When the skill triggers, Claude first prepares the project once (steps 0–6), then runs the
iteration loop (steps 7–10) for every challenger, and lets the user decide every promotion:

| Step | What happens |
|---|---|
| 0. Use case | Which decision the predictions feed, what errors cost, required model properties (e.g. a smooth, strictly monotonic price response for price optimisation) — written down as acceptance checks; how the decision is made today, with its documentation |
| 1. Data, stack, infrastructure | Short interview (local files and notebooks, Databricks and Delta tables, warehouse, …); available resources and their cost structure, and how the user weighs their own time against compute or cloud costs; critical feedback on the choices, above all on the iteration cycle length |
| 2. Data exploration | Potential (what is explainable beyond simple rules) and main challenges (cold starts, anomalies, short history, drivers unavailable at decision time, dependence) |
| 3. Evaluation design | Information cutoff from the use case; validation split chosen from the data (random, grouped, time-based, purged, spatial, distance matching, nested); disjoint selection and verification partitions |
| 4. Non-ML baseline | Today's method reproduced from its documentation, plus simple rule-based benchmarks — the reference for everything after |
| 5. Iteration 0 | The smallest model that meets the requirements; its run time is the cycle length; it is the first champion |
| 6. Exploratory shot | A quick, time-boxed model that addresses all exploration findings at once, purely for information gain and never promoted; compared with iteration 0 on the selection partition, it shows the potential, which findings carry signal and what still fails, and yields a ranked list of iteration candidates |
| **7–10. Iteration loop (repeated)** | One change per candidate with a stated hypothesis; per round a single challenger, parallel challengers or a bounded search (feature selection, tuning), chosen by expected information gain against run time and cost; parallel winners re-tested one on top of the other; acceptance checks as gates; promotion only if better in the selection partition and confirmed in the verification partition, with a block-bootstrap significance test; diagnosis before the next feature; the user promotes or rejects |

## When it triggers

Requests such as improving or tuning a model, adding or dropping features, comparing a new
model version with the current one or a baseline, asking whether a change is "really better",
setting up a backtest, cross-validation or train/validation/test split, translating a business
use case into model requirements, or diagnosing why a model loses to a simple benchmark.

## Repository layout

```
SKILL.md                     the binding rules of every step (loaded when the skill triggers)
references/
  preparation.md             steps 0–6: use case, interview, cycle length, exploration,
                             baseline, exploratory shot
  evaluation.md              information cutoff, split design, partitions, block bootstrap
                             (language-neutral algorithm), metrics, canonical methods
  iteration-loop.md          steps 7–10: one or several candidates, judging, results layout,
                             lessons from use
  diagnosis-and-pitfalls.md  how to diagnose a gap, worked examples, pitfalls
evals/                       test cases for maintainers, not part of the skill (see Testing)
```

`SKILL.md` stays short (about 11,000 characters) so that it remains loaded in full when a long
conversation is compacted. The references hold only what a frontier model would not reliably
do on its own: opinionated choices with their reasons, numbers from use, and incidents. They
are read when a step needs them. Agreements and the project's state live in the project's
notes, which the skill reads first.

## Installation

Claude Code looks for skills in `~/.claude/skills/<skill-name>/` (personal, all projects) and
in `<project>/.claude/skills/<skill-name>/` (one project). To use this repository as the
single source, link it instead of copying it, so edits here take effect immediately:

```bash
ln -s /path/to/ml_workflow_skill ~/.claude/skills/ml-model-iteration
```

If a copy already exists at that path, remove it first.

Skills do not sync across Claude surfaces. For claude.ai, upload the folder as a zip file
without `.git` and `evals/` under Settings > Features (the skill-creator's packaging script
leaves `evals/` out by itself); for the Claude API, upload it through the Skills
API. See [Agent Skills — where Skills work](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#where-skills-work).

## Testing

`evals/` holds test cases in the layout `claude plugin eval` reads: per case the user's
prompt, the expected outcome, one assertion per grader file, and input files. They check the
rules that a model would not follow on its own, for example planning a round after repeated
failures and judging a mixed result by the decision table. `SKILL.md` never points to
`evals/`, so an agent using the skill does not load it, and `evals/README.md` says the cases
are not instructions. Run them after every change to the rules, with a fresh agent that has
the skill or with `claude plugin eval . --scaffold` from the skill directory; keep the results
in a workspace next to the skill, not in this repository.

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

Used so far mainly on demand forecasting, predominantly in retail and energy; the lessons are
folded into the rules (`references/iteration-loop.md`, *Lessons from use*). Feedback from
other domains and from neural network models would be most welcome.
