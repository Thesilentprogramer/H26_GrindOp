# DEVELOPMENT.md

Judging-risk grill. Walk this before building.

Each item has three lines. **Lose** is how the entry fails judging. **Recommended lock** is the default. **Locked** is your decision. A recommended lock is not a decision until you fill `Locked:`.

This doc is ordered by what blocks judging. It has no file lists, timeboxes, or scan/trace/verify steps. [PRD.md](PRD.md) is the product contract. [CLAUDE.md](CLAUDE.md) is the assistant rules.

Three values this doc will not invent. Copy them from the organizer starter link and the MLH Gemma quickstart:

- Team name: ________
- Organizer starter URL: ________
- Gemma model id: ________

## 1. Submission gates

A public repo named `H26_TeamName_ProjectName`, forked from the official starter, with an OSI license and the seven README sections, is what gets judged. Cloud and paid APIs are forbidden. Every member must contribute. History must not be faked.

### 1.1 Repo identity

**Lose:** wrong name, not a fork, or private.

**Recommended lock:** Fork the organizer starter (URL left blank; do not guess it). Final name `H26_<TeamName>_ExitCapsule`. Fill team name and starter URL on the Locked line.

**Locked:**

### 1.2 License

**Lose:** no OSI license.

**Recommended lock:** Keep the starter's license if it already has MIT or Apache-2.0. If it has none, add MIT.

**Locked:**

### 1.3 README honesty

**Lose:** missing any of the seven handbook sections, or prose that describes unbuilt behavior.

**Recommended lock:** Create the seven headings early (problem, setup, usage, architecture, dependencies, AI model details, known limitations). Sentences inside a heading are written only after that part has actually run. Unbuilt work is listed under known limitations.

**Locked:**

### 1.4 Secrets and cloud

**Lose:** a key in git, or any paid or cloud API.

**Recommended lock:** `.env` gitignored, `.env.example` committed with empty `LLM_BASE_URL`, `LLM_API_KEY`, `LLM_MODEL`, `GITHUB_TOKEN`. Gemma comes only from the MLH quickstart or a local model. DigitalOcean stays out (Track 2).

**Locked:**

### 1.5 History

**Lose:** rewritten history, fake dates, or a member with no commit.

**Recommended lock:** Small imperative scoped commits. Each member commits work they reviewed. No history rewrite.

**Locked:**

## 2. Gemma 4 is real

Track 1 must use Gemma 4 meaningfully. The README must name model, version, source, access method, license, and role. The PRD still has the model id as `TODO(verify)`.

### 2.1 Identity

**Lose:** a guessed model id presented as fact.

**Recommended lock:** Copy id, base URL, and access method from the MLH Gemma quickstart onto the Locked line. Until that line is filled, the AI stage records `ai_suggested.status: "failed"` with reason `model id not verified` and still emits the observed sections.

**Locked:**

### 2.2 Meaningful use

**Lose:** Gemma unused, or one decorative call.

**Recommended lock:** One schema-validated call per flagged dependency. The prompt contains only observed evidence (risk indicators and usage snippets). The reply supplies replacement, alternatives, API mapping, migration steps, suggested tests, optional patch, and model-stated risks. Capsule metadata stores the model id, access method, and temperature `0.2`. Invalid JSON retries twice, then `status: "failed"`. No second model provider.

**Locked:**

### 2.3 Quota and failure

**Lose:** a hand-written section labeled as model output, or a hidden API error.

**Recommended lock:** Cache each response on disk under `.cache/`, one call per dependency, sequential. On failure, record it and continue. The demo shows a real call or an honest failure, with a screen recording only as fallback for a path that already worked once.

**Locked:**

## 3. Proof on LinkWatch

Checked against `rishrek/LinkWatch` at commit `f52f393` (shallow listing, not a clone):

- Runtime deps are in both `requirements.txt` and `pyproject.toml`. They disagree: `pydantic==1.10.13` vs `pydantic>=2`.
- `# PyYAML==5.1` is commented out in `requirements.txt`.
- `requirements-dev.txt` has `pytest` and `nose==1.3.7`.
- Makefile `test` is `python -m pytest`. `test-nose` is `nosetests tests/`. `PYTHON ?= python3.12`.
- Tests live in `tests/`: `test_cache.py`, `test_checker.py`, `test_config.py`, `test_report.py`.
- Python 3.12 and 3.13 only. Security bugs in this repo score zero.

### 3.1 Which files count

**Lose:** flagging a dep that is not declared, or hiding the pydantic split.

**Recommended lock:** Parse `requirements*.txt` and `pyproject.toml`. When a name is declared twice with different specs, record both as observed evidence. Commented lines are not dependencies.

**Locked:**

### 3.2 What "tests ran" means

**Lose:** a claimed pass from the wrong runner, or Docker presented as the result.

**Recommended lock:** Baseline is `python -m pytest` on Python 3.12. Try `nosetests tests/` only if pytest cannot collect, and label which command ran. If install fails (the pydantic split is a likely cause), record `not_executed` plus the stderr reason. Docker stays a limitation, not a result.

**Locked:**

### 3.3 Dynamic analysis

**Lose:** a hardcoded LinkWatch dep list or a capsule checked in before a run.

**Recommended lock:** No LinkWatch-specific branch in the tool. `examples/` is produced by running the tool, and the README names that command. Security findings are not the demo story.

**Locked:**

### 3.4 Minimum bar

**Lose:** no executed check and no reason.

**Recommended lock:** At least one dependency has a baseline run (`executed` or `not_executed` with a reason), and at least one model claim is checked in code (replacement exists on PyPI, and `hasattr` for a mapped API) with the result stored on that claim id. If time runs out, drop the migrated test rerun first. Keep the limitations section.

**Locked:**

## 4. Observed vs AI

This is the track rule judges will look for in the capsule and on the demo slide.

### 4.1 Where a checked claim lives

**Lose:** the model's replacement name copied into `observed`, or a test result stored under `ai_suggested`.

**Recommended lock:** Model output stays under `ai_suggested`, each claim id'd (`A0` replacement, `A1`… mappings). Code results live under `observed.verification.claim_checks` and point at `claim_id`. A compatibility gap is written by our code from the check result. `model_stated_risks` stay under `ai_suggested` even when they sound true.

**Locked:**

### 4.2 Report labels

**Lose:** a markdown file that mixes the two or fills a failed AI section by hand.

**Recommended lock:** Markdown is rendered only from the capsule JSON. Sections are labelled OBSERVED and AI-SUGGESTED as in PRD FR7. If `ai_suggested.status` is `failed`, that section shows the failure reason and stays empty of advice.

**Locked:**

### 4.3 Patches

**Lose:** a shell command or file write taken from model text.

**Recommended lock:** `proposed_patch` is data (`file`, `old_snippet`, `new_snippet`). Our code applies it only in a temp copy, and only when `old_snippet` matches the file exactly. Otherwise the migrated run is `not_executed` with reason `patch did not apply`.

**Locked:**
