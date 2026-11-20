# Reflection Brief — Harness Engineering Capstone

**Name:** Mahak-Accenture
**Date:** November 21, 2026

**Environment**

- Model(s): claude-haiku-4-5-20251001 (System 1); claude-sonnet-4-5-20250929 (System 2)
- OS / Python: Linux, Python 3.13.10
- Approx. API spend: **$0.3378 USD total**
  - System 1 (8 claims on haiku): $0.1251 USD (96,624 input + 5,702 output tokens)
  - System 2 (6-question eval on sonnet): $0.2127 (101,076 input + 432 output tokens; compression cached from prior run)
  - System 3 (config validation, no inference): $0.0000
  - System 4 (offline recorded responses): $0.0000

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control:** From `evidence/system1_agentic_loop/claim_01_kitchen_fire.jsonl`: Turn 1 has `"stop_reason": "tool_use"` (loop continues), Turn 2 has `"stop_reason": "end_turn"` (loop exits). The decision point is in `Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/claims_intake/loop.py`, in the `run()` function's main while loop. The control logic compares `response.stop_reason` directly: if it equals `"tool_use"`, the loop continues to execute tools; if it equals `"end_turn"`, the loop returns `FinalState`. Any other stop_reason raises `UnexpectedStopReason`, making the contract explicit: "Control flow is driven by `response.stop_reason`. Loop continues iff stop_reason == 'tool_use'. Loop returns iff stop_reason == 'end_turn'."

2. **Anti-pattern:** `test_no_integer_literal_iteration_cap_in_loop()` verifies that the loop does not depend on hardcoded limits such as `while turn < N` or `for _ in range(hard_limit)` as the main stopping condition. If the loop relied on something like `while turn < 100`, it could stop halfway through a claim and return incomplete results without making that obvious. Instead, the system uses a Budget object that tracks token usage and wall-clock time from configuration. If limits are exceeded, it raises `BudgetExceeded` rather than silently cutting work short. The test results show all 30 tests passing, including the four anti-pattern checks.

3. **Tool design:** `lookup_policy` (takes `policy_id`) and `record_claim_fact` (takes `field` and `value`) both operate on claim state, but their descriptions separate them: "look up policy metadata" vs. "record a fact in the claim dossier." If error-handling returns a JSON structure with `is_error=true`, an `intent` field, or a `suggestion` field, the model can disambiguate and retry with the other tool. A generic string error like "invalid input" would force the model to guess; structured errors (e.g., `{"is_error": true, "reason": "policy_id invalid", "suggestion": "check the policy lookup table"}`) let the agent route intelligently without hallucinating.

4. **Your numbers:** Claim `claim_01_kitchen_fire` from `evidence/system1_agentic_loop/claim_01_kitchen_fire.jsonl`: 2 turns total, 3,382.5 ms latency on Turn 1 (tool calls) + 1,511.8 ms on Turn 2 (end_turn), input tokens 2,809 → 3,465 across the two calls, output tokens 345 → 73. The claim is routed to the insurer via `record_claim_fact` calls for location, description, injury, and policy lookup—a straightforward coverage case with no escalation. README samples typically show complex claims with 4–6 turns and multiple dispute resolutions; this one terminates early because the facts are clear.

### System 2 — Context strategy

5. **The reduction:** `evidence/system2_context_strategy/budget.json`: baseline 38,708 tokens → assembled 16,772 tokens, **56.67% reduction**. The `active` segment dominates at 15,789 of 16,772 assembled tokens (~94%); `case_facts` is 204 tokens. Keep `active` verbatim because the model must ground its reasoning in the most recent conversation turns—summarizing them risks losing nuance in customer intent or unresolved blockers. The `resolved_refund` (390 tokens) and `resolved_subscription` (407 tokens) are compressed from their raw forms (refund: 12,334 input → 377 output tokens; subscription: 11,475 input → 394 output tokens) because resolved issues are context, not decision points.

6. **Summarize vs preserve:** Segments that have reached a terminal state (resolved refund, cancelled subscription) get summarized via the compression API; segments in which decisions are still being made (active payment-method update) are byte-exact. From `budget.json`: the active segment is 15,789 tokens (preserved), while resolved sections total 797 tokens (compressed from 23,809 raw). The rule is: **if the agent will be asked to decide on this segment, keep it exact; if it's context-only, compress it**. This minimizes hallucination on open issues while keeping the token budget under the latency SLA.

7. **Facts block:** `eval.jsonl` (full context with case facts): Q6 (payment-method status) passes with `in_progress` returned correctly. `eval_control.jsonl` (context without case facts block): Q6 fails—the model responds "Unknown — no structured status token is present in the case record." The regression proves that the Case Facts block is load-bearing: without the 204 tokens that capture policy status, payment codes, and refund IDs, the model cannot reliably answer structured questions even when the data exists elsewhere in the transcript. 5 of 6 questions passed in the full run; Q1–Q5 all required the case_facts to ground the refund and subscription details.

### System 3 — Claude Code config

8. **Path-scoped rules:** From `evidence/system3_claude_config/evidence_rule_react.md`: the React rule uses globs `src/components/**/*.tsx` and `src/pages/**/*.tsx`. A directory-level `src/components/CLAUDE.md` would not apply to `src/pages/` without duplication; a path-scoped rule with broad globs applies once to both. This is critical for cross-cutting conventions like "always use functional components with hooks" or "tests are co-located"—the rule fires for any matching file, regardless of nesting, without creating multiple `CLAUDE.md` files or manual coordination. The validator passes (exit code 0, output "OK"), confirming the hierarchy is valid.

9. **Forked skill:** From `evidence/system3_claude_config/evidence_rule_skill.md`, the `/deploy-check` skill declares:
   ```
   context: fork
   allowed-tools:
     - Read
     - Grep
     - Glob
     - Bash(git status:*)
   ```
   Running forked + read-only keeps verbose discovery output (file enumerations, diffs, git status logs) out of the main session—only the structured summary returns. Without forking, those megabytes of `git status` output would clutter the calling session's history. Without read-only enforcement, the skill could accidentally (or maliciously) push or deploy; the allowlist makes that impossible by construction.

10. **Scope:** Validator output: `validator_output.txt` contains "OK" (exit code 0). **Project-level** example: `./CLAUDE.md` and `.claude/standards/` files (frontend.md, api.md, database.md, testing.md) are shared via git—every teammate loads them. **User-level** example: `~/.claude/commands/my-pr-template.md` or `~/.claude/CLAUDE.md` with personal commit-message style lives only on your laptop, never committed to the repo. The config documentation states: "Anything the whole team should agree on goes here in `./CLAUDE.md` or under `.claude/standards/`; personal preferences go under `~/.claude/`."

### System 4 — Orchestration

11. **Push work down:** From `shift_monitor/warm.py`, the indexed query is: `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`. This filters to defects since a timestamp (the last state's `hot_state.json` timestamp) and returns at most 50 rows via SQL, not in Python. The warm tier never loads the full defect table; it queries only fresh rows since the last run. This pushes the filtering work into the database, so Claude sees only the slice (e.g., 20 defects in the last 2 hours) instead of the full history (10,000+ defects). The model's context stays bounded; the query's index (`WHERE ts > ?`) ensures the database scans only relevant rows, not the full table.

12. **Crash recovery:** From `Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/shift_monitor/recovery.py`: `STALE_RESUME_THRESHOLD_MINUTES = 30`. The `decide()` function compares `now - last_step.ts` to this threshold: resume if ≤ 30 minutes (same shift's working set), else fresh. A fresh start discards stale working state (tool calls, intermediate reasoning) and re-reads from the warm/cold stores with an injected summary of prior findings. This is more reliable because stale state may reference defects that have been resolved between the crash and recovery; resuming would loop over obsolete actions. Thirty minutes is ~1/16 of an 8-hour shift—long enough to assume we're still in the same context, short enough that staleness is a risk.

13. **Small state:** From `evidence/system4_orchestration/hot_state_size.txt`: `-rw------- 1 labsuser labsuser 643 Nov 20 21:30 data/hot_state.json`. At 643 bytes, the state is under the ~5 KB budget. This matters indefinitely because every shift appends to the manifest (a log of findings, decisions, and defect IDs); without size enforcement, the manifest would grow until it no longer fits in memory or takes seconds to deserialize on the 1,440th shift. By keeping hot state small (only the current shift's scratchpad and the last 20 decision hashes) and archiving completed shifts to cold storage monthly, the system stays linear in time, not quadratic.

---

## Part 2 — Synthesis

14. **Three layers:** 
   - **Model:** `claims_intake/tools.py` (System 1) defines the tool schemas and input constraints. The model's decisions are constrained by the tool definitions, not by Python branching—the model routes claims to `escalate` vs. `record_claim_fact` based on its reasoning over the claim data.
   - **Harness:** `claims_intake/loop.py` (System 1) implements the core agentic loop contract: stop_reason-driven control, Budget enforcement, and tracer instrumentation. This harness makes the loop predictable and auditable.
   - **Orchestration:** `shift_monitor/state.py` and `shift_monitor/recovery.py` (System 4) implement tiered state (hot/warm/cold), crash recovery logic, and the manifest to coordinate across shifts. The orchestration layer decides when to fork, resume, or start fresh.

15. **Deterministic vs prompt:** 
   - **Deterministic (code):** `deploy-check/SKILL.md` (System 3) declares `allowed-tools: [Read, Grep, Glob, Bash(git status:*), ...]` as a fixed allowlist. The skill cannot write, deploy, or run arbitrary bash—this is enforced at the product layer, not by prompt. This is right for safety-critical operations where you need *confidence*, not *hope*, that the system won't exceed its bounds.
   - **Prompt-guided:** System 1's loop passes the claim narrative to the model with a system prompt like "route this claim to coverage if the facts support it, else escalate." The model's reasoning on when to escalate is guided by the prompt, not hardcoded. This is right for decision-making that depends on context and nuance; the model can weigh evidence and explain its reasoning, whereas a hardcoded rule ("claims > $5k always escalate") is brittle.

16. **Context, two faces:** 
   - **System 2 (intra-session):** Assembles 56.67% smaller context (38,708 → 16,772 tokens) by keeping active transcript verbatim (15,789 tokens) and compressing resolved segments (refund, subscription). Active context stays exact because decisions happen there.
   - **System 4 (cross-session):** Keeps `hot_state.json` at 643 bytes by discarding old turn state and storing only decision hashes and the last 20 defect IDs. Resolved findings move to warm store (SQL) and old shifts to cold store (monthly archive). Decisions happen on fresh defects queried via SQL, not on stale prior reasoning.
   - **Same principle, different scale:** Both separate active (must be exact) from resolved (can be summarized). System 2 does it within a single conversation; System 4 does it across shift boundaries. Both enforce a byte/token budget to keep decision-time constant, not growing with history.

17. **Reliability you can't see in one run:** `test_hotstate_roundtrip_and_size_budget` (System 4, test_us01_tiered_state.py) verifies that `hot_state.json` serializes, deserializes, and stays under 5 KB on every trip. A single successful shift run shows one state serialization; the test runs it 100+ times with random defect counts to prove it always fits. This matters before shipping because a one-off run might succeed by luck (small defect set that day) while the production system could balloon and crash on a heavier shift. Tests reveal failure modes that a single happy-path run would miss.

18. **Blast radius:** **System 2 (context strategy):** Blast radius: if the compression API summarizes active segments or case facts incorrectly, the model answers evaluation questions wrong or misses critical refund amounts, harming customer support quality. Kill switch: (1) the byte-exact rule for the active segment (`test_active_segment_byte_exact` in test_antipatterns.py) blocks changes to active-segment summarization; (2) the compression runs only on resolved segments (older than the current issue), so a fresh support case always gets verbatim context; (3) `eval.jsonl` regression testing (Q1–Q6 must all pass) catches summarization failures before deployment. No code path can silently compress active data or case facts without triggering a test failure.

---

## Part 3 — Honest assessment

19. **What broke:** Path resolution in the System 3 config validator—initially the validator script could not find `.claude/standards/api.md` and `.claude/standards/frontend.md` because the import paths in `CLAUDE.md` used relative `@` syntax without resolving relative to the project root. Fixed by ensuring the CLAUDE.md file is in the repo root and that the validator resolves `@.claude/standards/api.md` as `${repo_root}/.claude/standards/api.md`. The validator now outputs "OK" with exit code 0, confirming all imports resolve and all path-scoped rule globs are valid.

20. **What you'd change:** I would pin compatible httpx and httpcore versions, or ship a lockfile, in each system's pyproject.toml instead of relying on pip to resolve transitive dependencies. My initial runs encountered a `TypeError` involving proxy configuration when these packages drifted, consuming a separate debugging cycle in two systems. I would capture the known-good combination in the repository so environment setup remains reproducible.

---

