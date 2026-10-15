# PRD: Dependency Exit Capsule Generator

**Event:** Hacktoberfest Hack Day Mumbai x ISTE KJSSE, 15 Oct 2026
**Track:** Track 1 - Gemma 4: Dependency Resilience
**Build window:** 10:30 AM to 4:30 PM IST (6 hours)
**Target repo:** https://github.com/rishrek/LinkWatch (Python 3.12 / 3.13 only)

## 1. Problem

Open-source maintainers depend on packages that become outdated, unmaintained, deprecated, or incompatible with newer Python versions. They find out when something breaks. Replacing a dependency is risky because nobody can easily verify that the replacement behaves the same, and AI suggestions alone are not trustworthy.

## 2. Product

A CLI tool that analyses a Python repository and, for every at-risk dependency, produces a reusable **Dependency Exit Capsule**: evidence-backed risk analysis, usage locations, a Gemma 4 proposed replacement and migration plan, executed test comparisons, compatibility gaps, and honest limitations.

**Core principle:** AI-generated suggestions are visibly separated from facts observed or verified by execution. The tool's value is in connecting the two.

## 3. Users

- **Primary:** open-source maintainers who want a safe, evidence-backed starting point for replacing a dependency.
- **Secondary:** contributors and reviewers who need to understand why a swap was proposed and what was tested.

## 4. Goals

- G1. Given a repo path or GitHub URL, automatically find at-risk dependencies using real metadata.
- G2. Show where and how each is used (file, line, API).
- G3. Use Gemma 4 meaningfully to propose a replacement, API mapping, migration steps, and tests for **every** flagged dependency.
- G4. Where feasible, run tests on original vs. migrated code and report differences.
- G5. Emit a reusable capsule (JSON + Markdown) with strict observed vs. AI-suggested separation.
- G6. Work live on LinkWatch with no pre-generated outputs.

## 5. Non-goals (cut for the 6-hour window)

Listed as future work in the README, not built:

- Web UI, URL-swap hosting, DigitalOcean deployment, job queue
- Docker sandboxing (we state this as a limitation; see Risks)
- Non-Python ecosystems
- Agent Skill / npx packaging
- Security vulnerability analysis (zero points per handbook)
- Fine-tuning or training any model

## 6. Functional requirements

### FR1 - Input handling
- Accept a local path or public `https://github.com/owner/repo` URL.
- Clone into a temp dir (shallow). Never modify the original.
- Record repo URL and commit SHA in the capsule.

### FR2 - Dependency discovery and risk scoring (observed)
- Parse `requirements*.txt` and `pyproject.toml` (use `tomllib`). Include pinned versions where present.
- For each dependency, collect:
  - PyPI: latest version, latest release date, classifiers (Python 3.12/3.13 support), `Development Status`, yanked flag, deprecation hints in description
  - Source repo (from PyPI project URLs) via GitHub API: archived flag, last commit/push date, open issue count
  - Version gap between pinned and latest
- Compute a risk score (0 to 100) from transparent rules in code, with each indicator listed as evidence. Rubric is documented in the README.
- Flag dependencies above a threshold as at-risk. Threshold is configurable.
- Handle network failures per-dependency without aborting the run.

### FR3 - Usage tracing (observed)
- Use `ast` to find imports (`import x`, `from x import y`, aliases) of each flagged dependency across the repo.
- Record file, line, imported name, and the call sites or attributes used.
- Map import names to distribution names (e.g. `yaml` to `PyYAML`) using installed metadata or a fallback table.
- Output a usage summary: total files, total call sites, most-used APIs.

### FR4 - AI migration analysis (ai_suggested)
- For each flagged dependency, send Gemma 4 the evidence: dependency details, risk indicators, usage sites with code snippets.
- Require structured JSON output validated against the schema:
  - `replacement` (name, rationale), `alternatives_considered`
  - `api_mapping`: list of `{old, new, notes}`
  - `migration_steps`: ordered list
  - `suggested_tests`: list of test ideas
  - `proposed_patch`: list of `{file, old_snippet, new_snippet}` edits (optional)
  - `risks`: model-stated concerns
- Retry on invalid output (max 2); on failure record `status: "failed"` and continue.
- Every claim gets an id so verification results can reference it.
- Record model name, version, access method, and parameters in capsule metadata.

### FR5 - Verification (observed)
All of this runs in a **temp copy**. Model output never runs as commands.

| Level | Check | When |
|---|---|---|
| L0 | Replacement exists on PyPI and its classifiers include Python 3.12/3.13 | Always |
| L1 | Create venv, install replacement, import it, `hasattr`-check each `api_mapping.new` | Always when install succeeds |
| L2 | Baseline: create venv with the repo's original deps, run the repo's test suite, record pass/fail/error counts | Whenever the repo has runnable tests |
| L3 | Migrated: apply `proposed_patch` in a temp copy, swap the dependency in a fresh venv, run the same tests, diff against baseline | Only when a patch applies cleanly |

- Each step records the exact command, exit code, duration, and truncated output.
- Unexecuted steps are recorded as `not_executed` with a reason (no tests, install failed, patch did not apply, timeout).
- Every subprocess has a timeout (default 300 s per step).

### FR6 - Comparison and gaps (observed)
- Diff baseline vs. migrated test results: newly failing, newly passing, unchanged.
- Report model-claim vs. reality mismatches (for example, a mapped API that does not exist) as compatibility gaps.
- List unresolved risks and limitations per dependency.

### FR7 - Capsule output
- One capsule JSON per dependency plus a combined summary, written to `out/`.
- Markdown report rendered from the JSON, with clearly labelled sections:
  - Dependency details, Risk indicators, Usage locations -> **OBSERVED**
  - Proposed replacement, Migration steps, Suggested tests -> **AI-SUGGESTED**
  - Executed test results, Verification of AI claims -> **OBSERVED**
  - Compatibility gaps, Open concerns and limitations
- Validate capsule JSON against the schema before writing.

### FR8 - CLI
```
python -m exitcapsule <repo-url-or-path> [--out out/] [--threshold 40]
                      [--stage scan|trace|ai|verify|report] [--no-verify]
```
Prints a concise terminal summary and the output paths.

## 7. Capsule schema (v0.1)

`exitcapsule/models.py` is the source of truth. Shape:

```json
{
  "schema_version": "0.1",
  "metadata": {
    "generated_at": "ISO-8601",
    "tool_version": "0.1.0",
    "target": {"repo": "url-or-path", "commit": "sha", "python": "3.12.x"},
    "model": {
      "name": "Gemma 4 (exact id TODO(verify))",
      "version": "TODO(verify)",
      "access_method": "MLH-provided hosted API | local",
      "license": "Apache-2.0 (per handbook; verify on model page)",
      "parameters": {"temperature": 0.2}
    }
  },
  "dependency": {"name": "", "pinned": "", "declared_in": ["requirements.txt"]},
  "observed": {
    "metadata": {"latest_version": "", "last_release": "", "py_classifiers": [], "repo_archived": false, "last_commit": ""},
    "risk": {"score": 0, "indicators": [{"id": "R1", "rule": "", "evidence": "", "weight": 0}]},
    "usage": {"files": 0, "call_sites": 0, "locations": [{"file": "", "line": 0, "symbol": "", "snippet": ""}]},
    "verification": {
      "claim_checks": [{"claim_id": "A1", "check": "hasattr", "result": "pass|fail|not_executed", "detail": ""}],
      "runs": [{"label": "baseline|migrated", "command": "", "exit_code": 0, "passed": 0, "failed": 0, "errors": 0, "duration_s": 0.0, "status": "executed|not_executed", "reason": ""}]
    },
    "comparison": {"newly_failing": [], "newly_passing": [], "unchanged": 0}
  },
  "ai_suggested": {
    "status": "ok|failed",
    "replacement": {"id": "A0", "name": "", "rationale": ""},
    "api_mapping": [{"id": "A1", "old": "", "new": "", "notes": ""}],
    "migration_steps": [""],
    "suggested_tests": [""],
    "proposed_patch": [{"file": "", "old_snippet": "", "new_snippet": ""}],
    "model_stated_risks": [""]
  },
  "compatibility_gaps": [{"source": "observed", "description": "", "related_claim": "A1"}],
  "limitations": [""]
}
```

## 8. Architecture

```
          repo path / URL
                |
         [ clone to temp ]
                |
   scanner  ->  tracer  ->  migrate (Gemma 4)  ->  verify  ->  report
  (observed)   (observed)  (ai_suggested)        (observed)   (JSON + MD)
                                  |
                         llm.py (retry, schema validation)
```

- Stages communicate through the typed models and can run independently from saved JSON.
- `llm.py` is a thin client over the endpoint documented by the MLH Gemma quickstart; model, URL, and key come from env vars.
- Prompts are versioned files in `prompts/`.

## 9. Tech stack

Python 3.12/3.13, `uv`, `pytest`, `httpx` or `requests`, `pydantic` (schema validation), `jinja2` (Markdown rendering), stdlib `ast`, `tomllib`, `subprocess`, `tempfile`. Model: Gemma 4 via the MLH-provided free access. Licenses of all libraries listed in the README.

## 10. Six-hour plan

| Time | Milestone | Owner |
|---|---|---|
| 10:30-11:00 | Clone LinkWatch, run its tests on 3.12/3.13, hand-list risky deps (ground truth), freeze `models.py`, create repo and license | All |
| 11:00-12:30 | Scanner + risk scoring | A |
| 11:00-12:30 | AST tracer | B |
| 11:00-12:30 | Gemma 4 client, prompts, schema validation | C |
| 12:30-1:00 | Wire pipeline end to end, first capsule | All |
| 1:00-1:30 | Lunch | |
| 1:30-3:15 | Verification L0-L3 and comparison | C + D (or B if 3 people) |
| 1:30-3:15 | Markdown/JSON report, CLI polish | B |
| 3:15-4:00 | Run on LinkWatch, fix bugs, README, limitations | All |
| 4:00-4:30 | Screen recording fallback, PPT (6 slides on organizer template), final push | All |

**Fallback order if behind schedule:** drop L3 first, then HTML view, then retries. Never drop the observed/AI-suggested separation or the limitations section.

## 11. Acceptance criteria

- [ ] `python -m exitcapsule https://github.com/rishrek/LinkWatch` runs end to end with no manual input.
- [ ] Every flagged dependency gets a capsule with all schema sections present.
- [ ] Risk indicators cite real fetched evidence.
- [ ] Usage locations match a manual grep spot-check on at least 2 dependencies.
- [ ] Gemma 4 output is schema-validated; failures are recorded, not hidden.
- [ ] At least one dependency has an executed baseline test run; migrated run is executed or marked `not_executed` with a reason.
- [ ] At least one AI claim is checked by code (L1) and its result appears in the report.
- [ ] Markdown report clearly labels OBSERVED vs AI-SUGGESTED.
- [ ] No API keys in git history; `.env.example` present.
- [ ] README has all required sections (see section 13).
- [ ] Open-source license file present; repo name is `H26_TeamName_ProjectName`.

## 12. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Gemma 4 model id, endpoint, or quota differs from assumptions | Verify in the first 30 minutes via the MLH quickstart; keep everything in env vars; cache responses |
| LinkWatch tests do not run on a clean env | Detect and record as `not_executed` with the reason; still deliver L0/L1 |
| Replacement is not a drop-in, so patches fail | Expected. Report as compatibility gap; this is the honest outcome the capsule exists to surface |
| Executing untrusted repo code without a sandbox | Run only in temp copies with timeouts; state clearly in limitations; recommend a container for untrusted repos; Docker is future work |
| Hosted API rate limits | On-disk response cache; one call per dependency; sequential calls |
| GitHub API rate limit | Optional token; cache; degrade gracefully when metadata is missing |
| Scope creep | Section 5 non-goals are binding |
| Cloud/paid services rule | Use only free MLH-provided Gemma access or local model; no other hosted services |

## 13. Required README sections (handbook)

1. Problem and solution overview
2. Setup instructions
3. Usage instructions
4. Architecture
5. Dependencies
6. AI model details: model name, version, source, access method, applicable license/usage terms, role of the model in the project
7. Known limitations

Also disclose any pre-existing code or third-party work, and mark incomplete features honestly.

## 14. Demo plan (live, on LinkWatch)

1. Show the real input: the LinkWatch URL.
2. Run the tool; show scan and risk output.
3. Show usage locations for one dependency.
4. Show Gemma 4's proposed replacement and mapping, labelled AI-SUGGESTED.
5. Show executed results: baseline vs. migrated tests, and any claim that failed verification.
6. Open the capsule (JSON + Markdown); point out the separation and the limitations section.
7. Keep a short screen recording ready as a fallback.

## 15. Open questions (resolve with organizers or docs early)

- Exact Gemma 4 model id, endpoint format, and quota via MLH.
- Does deploying anything on DigitalOcean violate the "no cloud services" rule? (Not needed for this scope.)
- Is the starter repo link provided, and what does it contain?
- Whether one project may be entered in both tracks.
- LinkWatch specifics (dependency list, test command, supported Python) are filled in during Phase 0 and should be recorded in the README.
