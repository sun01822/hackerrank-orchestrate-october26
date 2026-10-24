# Ghost Data Still Exists — Deletion-Audit Agent

HackerRank Orchestrate, October 2026. The agent audits whether a customer's personal data was really deleted. The full task is in [`problem_statement.md`](./problem_statement.md).

For each request in `dataset/deletion_requests.csv` the agent:

1. decides whether the person actually asked for deletion;
2. finds every copy of their data across databases, replicas, logs, exports, search, analytics, backups and documents;
3. judges the state at `audit_at`, using evidence rather than workflow claims;
4. writes `COMPLETE`, `INCOMPLETE` or `NOT_APPLICABLE`, with the ordered findings, remediation and verification steps.

It is a **hybrid agent**. A deterministic evidence pipeline does the investigation and planning. Claude reads only what code cannot: the intent of the request text (any language) and the admin-console screenshots.

## Results at a glance

| Measure | Value |
|---|---|
| Sample status accuracy (32 public samples) | **100%** |
| Step precision / recall / F1 | 98.4% / 97.4% / **97.9%** |
| Custom scenario cases | 5 / 5 exact |
| Regression tests | 14 / 14 pass |
| Full run (518 + 32 requests) | about 1 s, deterministic (byte-identical output) |
| LLM budget for a live run | about 13 calls, about 65k tokens, about $0.13 |

The submitted `output.csv` comes from an **offline run with 0 model calls**, because no API key was available. Details are in [`code/evaluation/usage_report.md`](./code/evaluation/usage_report.md) and [`code/evaluation/evaluation_report.md`](./code/evaluation/evaluation_report.md).

## Setup

You need:

- **Python 3.10+**
- **`pdftotext`** (poppler-utils), for the 4 PDF evidence files:
  - Debian/Ubuntu: `sudo apt install poppler-utils`
  - macOS: `brew install poppler`
  - Windows: install poppler and put `pdftotext` on `PATH`
- The **challenge dataset** in a `dataset/` folder next to `code/`. The dataset is not part of `code.zip`:

```text
<project root>/
├── README.md
├── code/
└── dataset/          ← deletion_requests.csv, sample_*.csv, evidence/, …
```

Install the one Python dependency (the Anthropic SDK):

```bash
python3 -m venv .venv
.venv/bin/pip install -r code/requirements.txt          # Windows: .venv\Scripts\pip install -r code\requirements.txt
```

To enable Claude, put an API key in an uncommitted `.env` file at the project root. The agent reads only the environment and `.env`:

```bash
echo 'ANTHROPIC_API_KEY=sk-ant-...' > .env
```

## Run

All commands run from the project root.

| Command | What it does |
|---|---|
| `.venv/bin/python code/main.py` | Audits every request, then writes and validates `./output.csv` (518 rows) and `code/evaluation/usage_report.md` |
| `.venv/bin/python code/main.py --offline` | Same, with no model calls: keyword intent classifier, screenshots skipped |
| `.venv/bin/python code/main.py --refresh-llm` | Ignores the answer cache in `code/.llm_cache/` and calls Claude live |
| `.venv/bin/python code/evaluation/main.py --write -v` | Scores the 32 samples and 5 custom scenarios, prints each mismatch, refreshes the metrics in `evaluation_report.md` |
| `.venv/bin/python -m pytest code/tests -q` | Regression tests (custom scenarios, lifecycle scoping, replica drift, serialisation, log parsing) |

`python3` works in place of `.venv/bin/python` for offline runs. Only the live path needs the `anthropic` package.

Every run ends with a **contract validator** that stops the run if any rule is broken:
- exact header, every `case_id` once in order, valid enums;
- consecutive `step_no`, phase order;
- `NOT_APPLICABLE` ⇔ `[]`;
- empty `evidence_id` on future steps;
- every cited ID exists and was captured at or before that case's `audit_at`.

## Approach overview

```text
dataset/ ─► ingest ─────────► Observation / Coverage ─► identity + intent ─► link to the request ─► state at audit_at ─► policy ─► plan + validate ─► output.csv
             SQL · JSON · logs   per file, per copy        email → user_id      tier 1: exact ids       latest post-request   rules →       ordered steps,
             TXT · PDF · PNG     (primary/replica)         Haiku: intent        tier 2: worker trail    observation per copy  treatment     cited evidence
             (Sonnet: PNG)
```

**1. Ingest.** One parser per evidence family. Each one emits the same `Observation` record:
- file and capture time, record time;
- system, resource, copy (primary, read, failover, reporting or regional replica);
- present or absent;
- identifier tokens (email, sha256 email hash, digits-only phone, `user_id`) and whether direct PII is still visible;
- state flags (`enabled`, `restore_allowed`).

Full collection dumps also emit a `Coverage` record, so a missing resource can be proven absent. The six log layouts (key=value, syslog, JSON-lines, bracketed, pipe-delimited, nested cloud export) share one tokenizer. Their lines are joined on `trace_id`, so a lookup key reaches the line that holds the result.

**2. Identity and intent.** The request email, current or historical, resolves to a `user_id` and every identifier that person ever held (`user_audit_log.csv`). Claude Haiku 5.5 classifies every distinct request text in two batched calls. Export, correction, opt-out and "don't delete" requests become `NOT_APPLICABLE`.

**3. Linking.** This decides which resources belong to *this* request.
- **Tier 1, exact identifiers:**
  - Sources: record fields, trace-joined lookup keys and data-catalog link events.
  - Look-alike decoy emails never match. A name alone never links.
- **Lifecycle scoping:** some people file several requests. Each request owns the evidence captured after the previous request's `audit_at`.
- **Tier 2, the deletion worker's trail:** this covers resources that carry no identifier, such as deleted accounts that answer 404 and fully anonymised rows. They are assigned by one global, exclusive assignment across all requests. It uses these signals:
  - the account-delete result agrees with the reported status;
  - the case's own downstream events fall 15–175 minutes after its account delete;
  - shared evidence files;
  - hints in the request text.

**4. State at `audit_at`.**
- Only files captured at or before `audit_at` are read for state or citations.
- Observations made before the request describe the pre-deletion world. They can link a resource but cannot prove a violation.
- The latest post-request observation of **each copy** decides, and a single surviving replica row makes the resource non-compliant.
- Workflow statuses and results are never used as proof.

**5. Policy.** `deletion_rules.csv` maps each resource's category to a treatment. The category comes from three sources, in order: the action the deletion workflow attempted, then a ledger's record type, then the system and record shape.

| Treatment | Applies to | Finding → remediation → verification |
|---|---|---|
| Delete | profile, search entry, export, cache, marketing profile, attachment, secondary identifier | `REPORT_REMAINING_RECORD` → `DELETE_RECORD` → `VERIFY_RECORD_ABSENT` |
| Redact | support message, email delivery log, account tombstone (keeps only the stable `user_id`) | `REPORT_EXPOSED_PII` → `REDACT_RECORD` → `VERIFY_RECORD_REDACTED` |
| Anonymise | analytics, telemetry | `REPORT_EXPOSED_PII` → `ANONYMIZE_RECORD` → `VERIFY_RECORD_ANONYMIZED` |
| Retain | security audit log, active legal hold | `VERIFY_RECORD_PRESENT` |
| Block restore | backup archive | `REPORT_RESTORE_RISK` → `BLOCK_RESTORE` → `VERIFY_RESTORE_BLOCKED` |
| Disable flow | integration / sync connector | `REPORT_ACTIVE_DATA_FLOW` → `DISABLE_DATA_FLOW` → `VERIFY_DATA_FLOW_DISABLED` |

**6. Plan.**
- Steps are ordered by phase, then the operation table (flows and restores are stopped before records change), then `system_id`, then `resource_id`.
- Verification covers the required end state of every relevant resource, not only the remediated ones.
- A finding cites the file that shows the violation.
- A `COMPLETE` verification cites the file that proves the compliant state.

**Why hybrid.** The grader checks exact resource IDs, evidence IDs and step order. Deterministic rules reproduce those exactly, cost nothing per case and can be tested. A per-case LLM agent would spend millions of tokens and give non-deterministic answers.

The LLM calls are built to stay cheap and safe:
- they are batched and de-duplicated;
- strict JSON-schema output, lowest thinking setting;
- every answer is cross-checked against the keyword classifier;
- answers are cached by content hash;
- a failed call is logged and skipped, never fatal.

## Project layout

| Path | Contents |
|---|---|
| `code/main.py` | Command-line entry point |
| `code/audit/agent.py` | `DeletionAuditor`, which wires the stages together |
| `code/audit/ingest/` | `sql.py`, `records.py` (JSON + TXT), `logs.py`, `documents.py` (PDF + screenshots), `normalize.py` |
| `code/audit/identity.py` · `linking.py` · `state.py` · `policy.py` · `planner.py` · `output.py` | One stage each |
| `code/audit/intent.py` · `llm/` · `prompts/` | Claude calls, cache, usage ledger, prompts (Markdown) |
| `code/evaluation/main.py` · `custom_cases.py` | Scorer and the five scenario datasets |
| `code/evaluation/evaluation_report.md` · `usage_report.md` | Accuracy, failures and fixes; token and cost report |
| `code/tests/` | Regression tests |

## Assumptions and known limits

- Rows are sorted by numeric `case_id`, matching the input file (IDs are unpadded: `case_33` … `case_550`).
- Files captured after `audit_at` may tell *which* resource belongs to a request. They are never used for its state or as a citation.
- Some expected records carry no identifier in any evidence and have no activity, so they cannot be linked without guessing.
- When several requests share a reported time slot, identifier-less resources can be indistinguishable. A deterministic tie-break decides, and the evaluation report quantifies how this affects the sample scores.
- In the offline submission, about 12 evaluation cases that are backed only by screenshots use activity citations instead of the screenshot's state.

## Packaging

```bash
zip -r code.zip README.md code -x "*/__pycache__/*" "*/.venv/*" "*/venv/*" "*/node_modules/*" "*/.git/*" "*/.DS_Store" "*/.pytest_cache/*"
```

The archive excludes `dataset/`, virtual environments, caches, nested archives, `.env` and any credentials.
