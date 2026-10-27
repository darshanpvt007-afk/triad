# Triad — GitHub Issue Triage Bot: Design

> Status: **design only, no code yet.** Decisions marked **[OPEN]** need an answer before building.

## 1. Problem

Maintainers of busy repositories spend hours sorting new issues: deciding what
kind of issue it is, how urgent it is, whether it is a duplicate, and whether the
reporter gave enough information. The work is repetitive, and slow triage drives
contributors away.

## 2. Goal

When an issue is opened, Triad posts a triage result within ~30 seconds:

| Output | Example |
|---|---|
| Type label | `bug`, `feature`, `question`, `docs` |
| Priority | `p0-critical` … `p3-low` |
| Missing info | "Please add your version and steps to reproduce" |
| Possible duplicates | "Possibly related: #142, #318" |
| Area | `area:api`, `area:ui`, … (configured per repo) |

### Non-goals (v1)

- Closing issues or editing issue text: Triad only suggests and labels.
- Triage of pull requests (possible v2).
- A web dashboard (possible v2; the GitHub UI is the interface for v1).

## 3. Principles

1. **Measure before building.** The evaluation harness and labeled dataset come
   *first*. Every classifier change is checked against them.
2. **The model judges, code decides.** The model answers small questions
   ("Is this a bug report?", "Does it include reproduction steps?"). Plain,
   testable code maps those answers to labels and actions.
3. **Swappable classifier.** One interface, many backends (TypeSafe, Claude,
   local model, keyword baseline). The baseline gives us a number to beat.
4. **Never silently wrong.** If a model call fails, Triad posts nothing and logs
   an error. A failure is never treated as "no".
5. **Safe by default.** Dry-run mode (log only, post nothing) is the default
   for every newly installed repo.

## 4. Architecture

```
GitHub ── issues.opened webhook (HMAC-signed) ──▶ FastAPI /webhook
                                                     │ verify signature
                                                     ▼
                                               job queue (v1: in-process
                                               background task)
                                                     │
                     ┌───────────────────────────────┼────────────────────┐
                     ▼                               ▼                    ▼
              Classifier interface           Duplicate finder        Repo config
        (typesafe | claude | local | baseline)  (embeddings +       (.github/triad.yml:
              → typed judgments              nearest neighbours)    areas, labels, dry-run)
                     └──────────────┬────────────────┘
                                    ▼
                        Decision rules (pure Python)
                                    ▼
                    GitHub API: add labels + one comment
```

### Components

| Module | Responsibility | Tested by |
|---|---|---|
| `webhook` | Verify HMAC, parse event, enqueue | Unit tests with recorded payloads |
| `classifier/` | Interface + backends, returns typed judgments | Eval harness |
| `rules` | Judgments → labels/comment (truth table) | Unit tests, table-driven |
| `duplicates` | Index past issues, return top-k similar above threshold | Eval harness |
| `github_client` | Labels, comments, issue search (GitHub App auth) | Mocked tests |
| `config` | Load per-repo `triad.yml` with safe defaults | Unit tests |
| `eval/` | Dataset builder + metrics report | Runs in CI |

## 5. Decision rules (sketch)

The model returns judgments; this table, in code, decides the outcome.

| is_bug | is_feature | is_question | Result |
|---|---|---|---|
| yes | – | – | `bug` |
| no | yes | – | `feature` |
| no | no | yes | `question` |
| unsure (0.3–0.7) on all | | | `needs-triage` (human decides) |

Missing-info check applies only to `bug`: ask for whatever of
{steps to reproduce, expected vs actual, version, environment} is absent.

Priority uses separate judgments (crash? data loss? security? workaround exists?),
combined in code, for example: security or data loss → `p0`.

## 6. Evaluation plan

- **Dataset:** ~300 closed issues from 2–3 large public repos that already label
  carefully (candidates: `fastapi/fastapi`, `pandas-dev/pandas`,
  `microsoft/vscode`). Map their labels to Triad's labels.
  Split: 200 for tuning, 100 **held out** (never looked at while tuning).
- **Duplicates:** use issues the maintainers closed as `duplicate` that link
  the original issue. These are the ground truth.
- **Metrics:** per-label precision/recall, macro-F1, duplicate recall@3,
  median latency, cost per issue.
- **Baseline:** keyword rules. Every backend must beat it on the held-out set.
- Results are written to `eval/results.md`. These numbers go in the README and on
  the CV.

## 7. Tech stack

- Python 3.12, FastAPI, httpx, pydantic, pytest, ruff
- GitHub App (not a personal token): installable on any repo, scoped permissions
- SQLite for the issue index in v1, Postgres later if needed
- Docker; deploy on Fly.io / Render / Railway free tier

## 8. Milestones (each ends with an exit test)

| # | Milestone | Exit test |
|---|---|---|
| M0 | Repo skeleton, CI, lint, tests | CI green on an empty test |
| M1 | Dataset builder + eval harness + keyword baseline | `make eval` prints baseline metrics |
| M2 | First model classifier backend | Beats baseline on held-out set |
| M3 | Decision rules + missing-info check | Table-driven tests pass |
| M4 | Duplicate finder | Recall@3 measured and reported |
| M5 | GitHub App webhook, dry-run mode | Real test repo issue → correct log output |
| M6 | Posting labels/comments, deploy | Live on 1–3 public repos |
| M7 | README with metrics, demo GIF, architecture diagram | Someone else can install it from the README |

## 9. Open decisions

1. **[OPEN] Classifier backend for M2:** TypeSafe (typed judgments with calibrated
   probabilities; needs a key, pricing unconfirmed) / Claude API / free local
   model. *Recommendation: build the interface so this can change, and start with
   whichever key you can get.*
2. **[OPEN] Standalone repo or a CodeSentinel module?** *Recommendation:
   standalone, so it is its own CV entry.*
3. **[OPEN] Which public repos for the dataset?**
4. **[OPEN] Hosting choice for M6.**

## 10. Risks

| Risk | Mitigation |
|---|---|
| Public repos label inconsistently | Pick well-maintained repos; inspect 30 issues by hand first |
| Model cost or limits | Cache results per issue; batch questions into one call |
| Spammy bot annoys maintainers | Dry-run default, one comment per issue, opt-in posting |
| GitHub API rate limits | Use App installation tokens; back off on 403/429 |
