---
name: inject-test-cases
description: Phase-2 transport only — push an ALREADY-APPROVED, signed-off test-case set into Azure DevOps under a parent PBI, mapping fields and Tags per the template and reporting created work-item IDs. Dumb transport, zero analysis. Use only when the user has reviewed a generated set and explicitly asks to inject / push / create the cases in Azure. Do NOT use to generate or edit cases, and do NOT auto-run on an unapproved set — if no signed-off set exists, send the user to analyze-pbi first.
---

# Inject Test Cases — Phase 2 Transport

Push an **already-approved** test-case set into Azure DevOps. This is **dumb
transport** — zero analysis, zero creative work. If the set has not been generated
and signed off via `analyze-pbi` (or `quick-test-cases`), stop and do that first.

**Argument:** the parent PBI ID to link the cases under → `$ARGUMENTS`
(If not provided, ask for the parent PBI ID before injecting.)

> This skill never invents, edits, or re-judges a test case. It maps the approved
> set to Azure fields and pushes it. All reasoning already happened in Phase 1.

## Procedure

1. **Confirm the parent PBI** — `$ARGUMENTS`. Do not proceed without it.
2. **Confirm sign-off & classification** — the set must be (a) the approved Phase-1
   output and (b) **classified `Automation` / `Manual`** by the Automation engineer
   (exactly one per case, 100% coverage). If anything is unapproved, return to
   `analyze-pbi`; if any case is unclassified, run the Automation classification pass
   first (see `CLAUDE.md` → Phase 2, step 5).
3. **Map fields** per `@.claude/context/test-case-template.md` — `test_type`,
   `scenario`, `impact_area`, `priority`, `execution_type`, and `Tags`
   (passed via the `tags` key per item) — `Tags` must include **exactly one
   execution-method tag (`Automation` / `Manual`)** per case. The MCP performs **no** tag
   judgement — it injects the decided tags verbatim and adds only the `Ai_MCP_Injected`
   provenance tag (do not include that one yourself); it dedupes.
4. **Inject the batch** — prefer `mcp__azure-devops__execute_qa_feedback` for the full
   approved set in one call. Use `mcp__azure-devops__create_english_test_case` /
   `mcp__azure-devops__create_arabic_test_case` only for individual cases or fallback.
5. **Ensure a Test Plan + Suite exist for the injected cases.** `execute_qa_feedback`
   links each Test Case to its parent PBI (`Tested By` reverse link) but does **not**
   create any Test Plan/Suite — without one, the cases are invisible in the Azure Test
   Plans UI even though they're correctly linked.
   - Determine the PBI's iteration path (sprint) from the PBI you just injected under.
   - `create_test_plan` is **not** documented as idempotent (unlike
     `ensure_bug_query_hierarchy` below) — calling it twice for the same sprint risks
     duplicate plans. **Ask the user once** whether a Test Plan already exists for this
     sprint before creating one; if they don't know or say no, call
     `mcp__azure-devops__create_test_plan(sprint_input=<iteration path>)`.
   - Single PBI: `mcp__azure-devops__create_test_suite_for_pbi(plan_id, pbi_id)` — a
     requirement-based suite that **auto-populates** from the `Tested By` links already
     created in step 4, no re-linking needed.
   - Multiple PBIs in the same sprint in one pass (e.g. injecting a whole sprint):
     `mcp__azure-devops__create_test_suites_for_sprint(plan_id, sprint_input)` creates
     one such suite per PBI in a single call — prefer this over calling the single-PBI
     variant in a loop.
   - Spot-check with `mcp__azure-devops__get_test_cases_from_suite(plan_id, suite_id)`
     that `total_test_cases` matches what you just injected.
6. **Provision the bug-query hierarchy** — call
   `mcp__azure-devops__ensure_bug_query_hierarchy(backlog_id=<PBI ID>)` once per PBI
   just injected. Idempotent — safe to call even if it already exists.
7. **Report back** — how many TCs were created and their Azure work item IDs, the Test
   Plan/Suite IDs (created or reused), and the bug-query hierarchy outcome
   (created/existing/error).
8. **Handle rejections** — if any case is rejected, fix the field that caused it and
   retry. **Never silently skip a case.**

## Optional follow-up

After injection, suggest auditing coverage and outcomes for `$ARGUMENTS`.
