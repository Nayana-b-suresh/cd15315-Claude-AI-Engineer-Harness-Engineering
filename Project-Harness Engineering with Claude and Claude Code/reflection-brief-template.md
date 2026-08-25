# Reflection Brief — Harness Engineering Capstone

**Name: Nayana Suresh**
**Date:24/08/2026**

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): claude-haiku-4-5-20251001
- OS / Python: Python 3.13.0, Linux 6.6.97+ x86_64
- Approx. API spend: $0.1018

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → In the trace runs/20260824_122740/traces/claim_02_stolen_bike.jsonl, the stop_reason sequence was tool_use → tool_use → tool_use → tool_use → end_turn across five turns. The continue-vs-stop decision is implemented in claims_intake/loop.py in run_agentic_loop: it continues when response.stop_reason == "tool_use" and returns when it equals "end_turn"; any other value raises UnexpectedStopReason. This makes the loop explicitly driven by the model's structured stop_reason rather than by a fixed number of turns. Artifact: runs/20260824_122740/traces/claim_02_stolen_bike.jsonl; implementation: claims_intake/loop.py.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → One anti-pattern checked by tests/test_antipatterns.py is using an integer-literal iteration cap as the primary stopping mechanism, such as for _ in range(5) or while turns < 5. The test test_no_integer_literal_iteration_cap_in_loop ensures the loop instead uses the model's stop_reason and configured budgets. If the loop used a fixed iteration cap, a claim that required more turns could stop before the model returned end_turn, causing the claim to remain incomplete or preventing the agent from completing the required tool calls. My System 1 test suite passed 29 tests, confirming the anti-pattern checks passed.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → Two tools with related inputs are lookup_policy and record_claim_fact, but their descriptions clearly separate their jobs. lookup_policy says to use it early to confirm the policy, coverage, and deductible, while record_claim_fact is specifically for recording one normalized fact from the claimant's statements, one fact per call. Their input schemas also prevent confusion: lookup_policy requires policy_id, whereas record_claim_fact requires field and value. The structured tool error in claims_intake/tools.py includes is_error, error_category, is_retryable, and message, allowing the agent to distinguish a permanent error from a retryable failure instead of treating every error as a generic string. Artifact: claims_intake/tools.py, lines 32–63 and 198–206.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → For claim_02_stolen_bike, the run took 5 turns and had an estimated cost of $0.0206, with an outcome of routed. The run artifact is runs/20260824_122740/traces/claim_02_stolen_bike.jsonl, and the overall run reported a total estimated cost of $0.1018. This differs from the README's sample because the number of turns and tokens depends on the model's actual tool-use decisions and responses for each claim, so live API runs are not expected to reproduce the exact sample numbers. Artifact: runs/20260824_122740 and its claim summary.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → In run runs/20260824-124026, the baseline transcript was 38,708 tokens and the assembled context was 16,770 tokens, giving a 56.68% reduction. The active section dominated the assembled context at 15,789 tokens, compared with 204 tokens for case_facts, 394 for resolved_refund, and 401 for resolved_subscription. The active segment is kept verbatim because it represents the current unresolved conversation and changing it could lose exact details needed to answer the user's current question. Artifact: runs/20260824-124026, including the run output and budget artifact.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → The rule is to summarize older resolved conversation segments while preserving the active segment byte-exact. In my run, resolved_refund was reduced to 394 tokens and resolved_subscription to 401 tokens, while the active segment remained 15,789 tokens. The case_facts block was separately extracted into a compact 204-token structured representation. This keeps historical information useful without unnecessarily spending context on old resolved turns, while protecting the exact current conversation. Artifact: runs/20260824-124026.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → In the normal evaluation, Q1 and Q6 both passed, but in the case-facts-stripped control, Q6 failed as expected while Q1 unexpectedly passed. Q6 asked for the exact structured status token of the payment-method update issue, and without the case-facts block the system could no longer provide that exact status. The unexpected pass on Q1 shows that the refund amount was still recoverable from the remaining context, so removing case facts did not affect every question equally. This demonstrates that the structured facts block preserves critical information that may not be reliably available after context compression. Artifact: runs/20260824-124026, [eval-control] section: Q1 UNEXPECTED PASS, Q6 FAIL (expected FAIL), with 6/6 passed in the normal evaluation.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → The rule file .claude/rules/react.md uses path-scoped frontmatter with paths: - "src/components/**/*" - "src/pages/**/*". This means the React conventions load only when editing files under those paths, rather than applying to an entire directory through a broad CLAUDE.md. This is better for cross-cutting conventions because the rules are targeted to the file types/areas where they actually apply, reducing irrelevant instructions in other parts of the project. The artifact also states that it “Loads when editing any file under src/components/ or src/pages/,” confirming the intended scope.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → The skill defines context: fork and an allowed-tools list containing only Read, Grep, Glob, and read-only Git/GitHub commands such as Bash(git status:*) and Bash(gh pr checks:*) (.claude/skills/deploy-check/SKILL.md). Running it forked keeps the verbose discovery output out of the main session, returning only the structured pass/fail summary. The read-only allowlist prevents the skill from modifying files, pushing, deploying, or running migrations. Without the fork, the intermediate output would pollute the main session; without the read-only allowlist, the deploy-check could have a much larger and potentially dangerous side effect.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The validator/config distinguishes project-level and user-level scope. A project-level example is ./CLAUDE.md, along with .claude/standards/ and .claude/rules/; these are stored in the repository and shared with the team. A user-level example is ~/.claude/CLAUDE.md or ~/.claude/skills/, which contains personal preferences and is not shared through version control. This distinction is documented in CLAUDE.md, lines 9–12.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The indexed query returned 12 defects from a warm tier containing 48 defects. The indexed query is get_recent_defects. The model never sees the full history because the database/query layer filters the historical records first and sends only the relevant results to the model. This reduces context size and keeps retrieval deterministic.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → The system resumes only when there is an incomplete state with steps and the most recent step is 30 minutes old or less (STALE_RESUME_THRESHOLD_MINUTES = 30). If there are no steps, the state is complete, or the latest step is older than 30 minutes, it starts fresh. A fresh start can be more reliable because a stale partial state may no longer represent the current shift's working set; restarting with the findings already captured in the manifest injected as a summary preserves useful information without blindly continuing stale execution.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → My hot_state.json was approximately 1.8 KB. The system keeps hot state under the ~5 KB budget. This matters because the system runs once per shift indefinitely; without a strict size limit, state would continuously grow and make future runs more expensive, slower, and harder for the model to reason over.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: claims_intake/system_prompt.py — defines the model's role and decision-making behavior.
    → Harness: claims_intake/loop.py and claims_intake/tools.py — the loop controls stop_reason, while tools enforce structured operations and errors.
    → Orchestration: System 4's orchestration/recovery artifacts — these manage work across runs and keep persistent state small.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → A deterministic behavior is the terminal-loop rule in claims_intake/loop.py: the loop continues only when stop_reason == "tool_use" and returns on end_turn. A prompt-guided behavior is the model deciding when to classify, assess severity, route, or escalate. Code is appropriate when correctness must be guaranteed; prompts are appropriate when the decision requires reasoning over the claim.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 manages context within a session, reducing the approximately 47,000-token baseline by at least 50% through summarization and selective preservation. System 4 manages context across sessions, keeping hot_state.json at approximately 1.8 KB, below the ~5 KB budget, while older information remains in the warm tier. Both use the same principle: keep only the information needed for the current task, but System 2 manages token context while System 4 manages persistent state.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → tests/test_antipatterns.py guarantees that the loop does not use magic-string matching or a hard-coded integer iteration limit and that stop_reason controls termination. A single successful run might still appear correct even if the implementation had one of these fragile anti-patterns. Tests matter because they prevent the behavior from silently regressing later.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → I would pick the claims-intake system. If the model misbehaves, the main risk is an incorrectly classified or incorrectly routed claim. The blast radius is limited by the tool layer: tools.py validates inputs, requires classification and severity before routing, and uses structured errors. The terminal routing/escalation tools also prevent repeated terminal actions. The loop's stop_reason handling is another enforcement point.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → The first attempt failed because I used the wrong run-directory timestamp/path when searching the traces. I corrected it by checking the actual runs/20260824_122740 directory and then inspecting the generated JSONL traces.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would make the workflow easier to inspect by providing a single command or report that summarizes the important run artifacts—turn counts, stop reasons, costs, and state size—instead of requiring several separate grep/find commands. The implementation itself benefits from keeping orchestration state and model reasoning separate.
