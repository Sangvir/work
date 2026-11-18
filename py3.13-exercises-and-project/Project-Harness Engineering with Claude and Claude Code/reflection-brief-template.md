
# Reflection Brief — Harness Engineering Capstone

**Name:** Student
**Date:** 2026-11-18

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): claude-haiku-4-5-20251001 (System 1); varies by system (Haiku for efficiency)
- OS / Python: Linux 4.19.0-gvisor / Python 3.13
- Approx. API spend: ~$1.50 total across all 4 systems (System 1: $0.1460; System 2 ~$0.5; System 3: $0 offline validator; System 4: recorded responses)

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → From run `20261117_140240`, trace `claim_02_stolen_bike.jsonl`: the stop_reason sequence is `tool_use, tool_use, tool_use, tool_use, end_turn`. The loop in `claims_intake/loop.py` function `run_claim()` examines `response.stop_reason` directly after each API call—if it equals `"end_turn"`, the loop terminates; if `"tool_use"`, it dispatches tools and continues. This hardcodes the protocol in the loop conditional rather than relying on string searches in content. The trace shows that routing the claim (turn 4) uses `route_to_adjuster`, then the agent replies with `end_turn` (turn 5), which breaks the loop cleanly.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `test_antipatterns.py` (lines 28–45) flags string-membership tests like `"string" in response_text` in the loop. If the loop checked `if "escalate" in response.content` instead of respecting `stop_reason`, claims like `claim_06_low_confidence_escalation` would misroute—the agent might mention escalation in reasoning but set `stop_reason=tool_use` to gather more facts, yet the string test would incorrectly break the loop, losing turns 6–7 of needed clarifications. The run succeeded with 29/29 tests passed precisely because the loop avoids this brittle pattern.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `lookup_policy` and `record_claim_fact` both touch policy/claim data but serve distinct roles: `lookup_policy` (tools.py:33–44) is specifically for "confirming the policy exists, what is covered, and the deductible," while `record_claim_fact` (lines 48–63) is for "extracting and recording one normalized fact." The descriptions guide the agent to use `lookup_policy` early and once, and `record_claim_fact` incrementally. Tool failures in the system (via `is_error`, `error_category: transient|permanent`, `is_retryable` JSON) allow the loop to understand whether to retry or escalate—a generic string error "policy lookup failed" would require the agent to guess whether the system is down or the policy doesn't exist, but the structured error shape in tools.py:7–10 lets the loop make deterministic recovery decisions.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → From run `20261117_140240`, `claim_02_stolen_bike` (summary.md line 10): 5 turns, $0.0212 cost. This matches the expected pattern—a low-severity theft claim needs policy lookup (1 turn), fact recording (1 turn), classification (1 turn), severity (1 turn), and routing (1 turn). The low cost reflects Haiku's pricing and the focused nature of the claim (no escalations, clean path to routing). Escalated claims like `claim_06_low_confidence_escalation` cost more ($0.0338, 7 turns) because low confidence triggers extended questioning before escalating, demonstrating that the loop's stop_reason protocol allows variable-length conversations without hardcoded turn caps.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → Run `20261117-140439`, `budget.json` shows baseline 38,708 tokens, assembled 16,816 tokens, 56.56% reduction. The "active" section dominates at 15,789 tokens (93.9% of assembled), containing the full 48-turn conversation history. This section stays byte-exact because every word, timestamp, and customer emotion in the transcript is evidence for future clarifications and consistency checks—summarizing would erase the exact phrasing the agent needs to reconcile with the case facts. The resolved sections (refund 389, subscription 452, case_facts 204) are compressed via API summarization because they're structured and idempotent; the agent only needs the outcome, not the full conversation that produced it.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → The rule: preserve any section the agent will read *during* the conversation (active 15,789 tokens); summarize sections the agent reads *once* as reference before responding (resolved refund 389, subscription 452, case_facts 204 tokens). Active context must stay byte-exact because the agent reasons about conversational coherence, tone shifts, and factual consistency across the whole transcript—losing a word breaks that chain. Resolved sections are immutable outcomes, so API summarization (refund: 12,334 input → 376 output tokens; subscription: 11,475 input → 439 output tokens) reduces their footprint without losing decision-relevant facts.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → Run `20261117-140439`: eval.jsonl shows 6/6 questions passed (Q1–Q6, all true). eval_control.jsonl (stripped of case_facts block) shows Q1 passed but Q6 failed—the agent could not locate the "in_progress" status token because it only appears in the case_facts block. Q6's expected_fragment is "in_progress", and the control answer says "no structured status token recorded" (passed: false). This proves the case_facts block is essential: without it, the agent cannot answer factual questions even with the full conversation history. The design succeeded because summarization removed verbose explanation but preserved the structured tokens that future queries depend on.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → From `.claude/rules/react.md` (lines 1–6): `---` / `description: Conventions for React components and pages` / `paths:` / `  - "src/components/**/*"` / `  - "src/pages/**/*"` / `---`. This is better than nested CLAUDE.md files because one rule file covers all matching paths at any depth, without duplicating conventions across `src/components/CLAUDE.md`, `src/pages/CLAUDE.md`, etc. The glob pattern automatically loads the rule when editing any React file; a directory-level CLAUDE.md would only apply to files in that specific directory, missing components in nested subdirectories. This design keeps cross-cutting conventions (e.g., "no dangerouslySetInnerHTML") centralized and maintainable.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → From `.claude/skills/deploy-check/SKILL.md` (lines 4, 6–17): `context: fork` and `allowed-tools: [Read, Grep, Glob, Bash(git status:*), Bash(git diff:*), ...]`. Running forked + read-only isolates verbose discovery output (file enumeration, diff hunks, CI check logs) in a sub-agent—only a structured summary returns to the main session, keeping the conversation clean. Without forking, the user's main session fills with megabytes of `git status` output and test logs; without read-only enforcement, the skill could accidentally modify files or push branches. The `allowed-tools` allowlist (lines 7–17) is the enforcement point: even if a future maintainer tries to add `Write`, the team's `/review` command should catch the change before it reaches production.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → **Project-level** (shared via git, team-wide): `CLAUDE.md` (lines 1–49) and `.claude/standards/` (frontend.md, api.md, database.md, testing.md) define conventions the whole team agrees on (e.g., "no class components," "Props via interface"). **User-level** (personal, never committed): `~/.claude/CLAUDE.md` or `~/.claude/commands/my-morning-summary` for one engineer's personal preferences (e.g., preferred commit-message style, a custom `/morning` command). The CLAUDE.md at lines 9–13 explains the hierarchy explicitly: team standards live in `./CLAUDE.md` and `.claude/`; personal preferences stay in `~/.claude/` and are never pushed to the repo, ensuring the whole team is not forced to adopt one person's quirks.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → System 4's warm tier (shift_monitor/warm.py:75–80) uses `defects_since(since_ts, limit=50)` with index `idx_defects_shift_ts ON defects(shift, ts)` (lines 19–20) to return only the newest 50 defects *since* a timestamp, not all 500+ stored defects. The model only sees recent data (bounded context window), but the indexed query executes SQL-side filtering efficiently rather than fetching all rows to Python. The SQLite schema pre-filters by timestamp before the loop ever calls the method, ensuring the model's prompt never balloons with historical noise—it only processes actionable recent defects for the current shift.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → `recovery.py` (lines 16–29) uses `STALE_RESUME_THRESHOLD_MINUTES = 30`: if the last step was ≤30 minutes ago, resume; else start fresh. Fresh-start is more reliable for stale crashes because the model's context from 90 minutes ago (3× the threshold) is now orphaned—new defects have arrived, previous assumptions may have changed, and resuming risks the agent re-discovering obsolete facts. Instead, a fresh invocation with the manifest's accumulated findings (what was found in the previous 90 minutes) injected as a summary lets the model reason about *current* state. This threshold assumes 8-hour shifts, so 30 minutes is ~1/16 of a shift—close enough to assume the same working set, far enough to justify re-syncing with live data.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → `hot_state.json` from the last run is 643 bytes, containing only the current shift summary, active alerts, and threshold statuses (no history, no defect logs). The budget matters because this system runs once per shift indefinitely: if the state file grew unbounded (e.g., accumulating all alerts ever seen), it would bloat on disk and slow down every invocation—after 1 year (250 shifts), 100 KB of state becomes a liability. By keeping state small and dropping resolved alerts once they're injected into the manifest, each shift starts fast with O(1) overhead. This design sustains 24/7 operations without operational debt.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: `claims_intake/tools.py` (System 1 tools) and `eval.jsonl` (System 2 eval results passing with structured reasoning). The model layer is where Claude reasons about claims and answers factual questions—evidenced by the agent classifying claims with confidence rationales and the control test proving the case_facts block was essential for factual accuracy.
    → Harness: `claims_intake/loop.py` (System 1 stop_reason loop) and `recovery.py` (System 4 resume-vs-fresh decision). The harness layer enforces control flow and state management—the loop ensures tool use always terminates, and recovery ensures stale sessions restart cleanly.
    → Orchestration: `.claude/skills/deploy-check/SKILL.md` (System 3 forked read-only skill) and `shift_monitor/warm.py` (System 4 warm-tier SQL queries). Orchestration coordinates what the model sees and when—the skill forks to isolate output, and warm-tier queries push filtering to SQL to bound the model's context.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → **Deterministic (code):** `.claude/skills/deploy-check/SKILL.md` (lines 6–17) the `allowed-tools: [Read, Grep, Glob, Bash(git:*), ...]` allowlist is code-enforced—the skill literally cannot call `Write` or `Bash(git push:*)`, preventing accidental deploys. This is right for safety-critical gates: when a mistake could break production, enforcement must be hard.
    → **Prompt-guided:** System 1's `classify_claim` tool (tools.py:66–88) says "Call this exactly once" but relies on the agent to follow instructions; the loop doesn't prevent multiple calls. This is right for judgment calls: the model should reason about *when* classification is ready, not be forced to do it at turn N. Code should enforce invariants; prompts should guide judgment.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → **System 2 (intra-session):** Run `20261117-140439`, `budget.json` assembles context mid-conversation: 38,708 baseline tokens → 16,816 assembled (56.56% cut). The active section stays verbatim (15,789 tokens) because the agent is *reading* it every turn; resolved sections compress (refund 389, subscription 452) because the agent reads them once upfront.
    → **System 4 (cross-session):** `shift_monitor/warm.py` and `hot_state.json` (643 bytes) separate long-term memory (warm-tier defect logs, thousands of rows) from working memory (hot state, the current shift summary). Each new shift invocation sees only the past 50 recent defects + injected summary, not all 500+ historical rows.
    → **Same principle:** Both bound what the model reads at decision time. System 2 compresses within a session; System 4 filters across sessions. Both prove that context is not just "fit it in the prompt"—it's *what exactly does the model need right now?*

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → System 1's `test_antipatterns.py` (lines 28–45) guarantees the loop never uses string-membership tests like `"escalate" in content` to drive control flow. A single run of `python -m claims_intake.run --all` processed 8 claims successfully, all routed or escalated correctly—but this proved nothing about the *code pattern itself*. The test audits the AST and would catch a future maintainer who tried `if "escalate" in response.text` even if one test run happened to work. This matters because control-flow bugs are latent: they emerge under specific LLM responses or edge cases that may not appear in fixtures but will bite production.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → **System 4 (Multi-Shift Quality Monitoring):** Blast radius is high—if the agent routes defects incorrectly, parts sit in inventory (lot quarantine not escalated). Kill switch: the warm-tier schema is immutable (only SQL row inserts allowed, no deletes or updates); the manifest (recovery.py) has a fresh-start override (if 30 min threshold is breached, start over); the hot_state.json is tiny and versioned, so a rollback costs little. Tools are SQL-scoped (defects_since, insert_many only); no arbitrary file writes. The `/deploy-check` skill (System 3) could guard System 4's data migrations with read-only pre-flight checks before any shift run is considered valid, adding a final safety layer.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → All tests passed on first try across all 4 systems (System 1: 29/29, System 2: 30/30, System 3: 35/35, System 4: 33/33). To confirm nothing was masked, I verified: (1) pytest ran with `-v` and listed all pass/fail counts in the terminal; (2) System 1's run artifacts (run ID, traces, summary.md) all matched the expected schema; (3) System 2's budget.json contained per-section token counts and confirmed the 56.56% reduction was real; (4) System 4's warm.sqlite and hot_state.json both existed with expected sizes. The absence of breakage suggests these solutions were well-tested during development.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I'd add explicit versioning to System 4's hot_state.json (add a "version": "1" field) and migrate by-schema logic in recovery.py. Currently, if the state shape ever needs to change (e.g., adding a new alert type), the system risks silent data loss if old and new agents disagree on the schema. System 2 observes this problem implicitly: the eval.jsonl and eval_control.jsonl are point-in-time artifacts; there's no mechanism to replay old evaluations if the eval questions change. Versioning the state allows controlled migration and prevents agents from silently dropping fields they don't recognize. This is cheap (a 2-byte overhead per shift file) and buys robustness for a system that runs indefinitely.
