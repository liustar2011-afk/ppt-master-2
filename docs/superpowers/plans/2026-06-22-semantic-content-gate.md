# Semantic Content Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prevent sparse or semantically incomplete presentation pages from reaching PPTX export.

**Architecture:** Strategist writes an optional but pipeline-required `analysis/page_contract.json` for newly created projects. Executor maps visible SVG groups to the contract through `data-role` and `data-ref`. A new checker validates required semantic units, relation rendering, and estimated content-region coverage before post-processing. Existing projects remain compatible unless the gate is explicitly required.

**Tech Stack:** Python standard library, JSON, XML ElementTree, existing PPT Master SVG workflow.

## Global Constraints

- Use only the Python standard library.
- Do not add automated tests or a `tests/` directory.
- Preserve existing project compatibility when `analysis/page_contract.json` is absent.
- Treat missing required semantic units as export-blocking errors.
- Treat density as a warning by default; make it an error only when a page contract marks it required.

---

### Task 1: Define the page-contract and SVG annotation contract

**Files:**

- Create: `skills/ppt-master/references/content-quality-gate.md`
- Modify: `skills/ppt-master/references/strategist.md`
- Modify: `skills/ppt-master/references/executor-base.md`

**Interfaces:**

- Produces: `analysis/page_contract.json` with `version`, `pages[]`, `required_refs`, `required_relations`, and optional `density` fields.
- Consumes: top-level SVG groups marked with `data-role` and `data-ref`.

- [ ] **Step 1:** Add the JSON schema, role vocabulary, and severity rules to the reference document.
- [ ] **Step 2:** Require Strategist to write one page contract record per output page after the content outline is locked.
- [ ] **Step 3:** Require Executor to map visible page groups to contract references before the SVG quality gate.

### Task 2: Implement the semantic and density checker

**Files:**

- Create: `skills/ppt-master/scripts/page_content_checker.py`

**Interfaces:**

- Consumes: `<project>/analysis/page_contract.json` and `<project>/svg_output/*.svg`.
- Produces: console report and non-zero exit code when semantic errors exist.
- CLI: `python3 scripts/page_content_checker.py <project> [--require-contract]`.

- [ ] **Step 1:** Parse and validate the contract schema without third-party dependencies.
- [ ] **Step 2:** Read SVG elements recursively, collect `data-ref` / `data-role`, and calculate approximate coverage within the declared content region.
- [ ] **Step 3:** Report missing pages, missing references, missing relations, duplicate references, and threshold failures with page-scoped severity.
- [ ] **Step 4:** Return exit code one for errors and zero for warnings-only reports.

### Task 3: Wire the gate into the PPT pipeline and scripts documentation

**Files:**

- Modify: `skills/ppt-master/SKILL.md`
- Modify: `skills/ppt-master/scripts/README.md`
- Create: `skills/ppt-master/scripts/docs/page_content_checker.md`

**Interfaces:**

- Pipeline placement: after `svg_quality_checker.py`, before notes and post-processing.

- [ ] **Step 1:** Add the contract-generation and annotation requirements to the relevant workflow steps.
- [ ] **Step 2:** Add `page_content_checker.py` as a mandatory export gate for new projects and a compatibility-safe warning for legacy projects.
- [ ] **Step 3:** Add usage and migration examples to script documentation.

### Task 4: Smoke-verify with the active project

**Files:**

- Create: `projects/安全能力可信核验服务场景建设方案_ppt169_20260622/analysis/page_contract.json`

**Interfaces:**

- Produces: a deliberate error report on currently sparse SVG pages, proving the checker detects the original failure mode.

- [ ] **Step 1:** Create a minimal contract for selected pages that includes relationships and evidence fields omitted from current SVGs.
- [ ] **Step 2:** Run the checker and verify it reports page-scoped semantic errors.
- [ ] **Step 3:** Run the checker's help command and Python compile check.

## Self-Review

- Page IR, semantic mapping, content coverage, and export gate requirements are covered by Tasks 1 through 3.
- Existing project compatibility is covered by the absent-contract warning behavior in Task 2.
- The active project supplies the operational regression sample in Task 4.
