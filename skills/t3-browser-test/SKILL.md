---
name: t3-browser-test
description: >-
  Spawn an independent testing agent through T3 Code orchestration (`delegate_task`) to inspect the
  current changes and test them in T3's real collaborative browser preview, then report reproducible
  browser evidence and retest fixes. Defaults to Codex GPT-5.6 Luna and accepts any provider/model
  from `orchestrator_capabilities`. Use when the user asks to test changes in a real browser,
  browser-test a branch or diff, run an independent UI tester, verify a frontend fix in T3, dogfood
  changed behavior, or retest browser failures, and the session runs inside T3 Code. This skill keeps
  the spawned tester read-only with respect to source code; the current agent remains responsible
  for fixes.
---

# Browser testing via a T3 agent

Act as the **implementer/coordinator**. Spawn a fresh **browser tester** as a T3 delegated task in
the current checkout. The tester inspects the actual diff, operates T3's collaborative browser
preview, and reports evidence; it never edits source.

## 1. Discover the T3 orchestration tools

The tools come from the `t3-code` MCP server. Names may carry a harness prefix
(`mcp__t3-code__delegate_task`, `mcp__t3_code__delegate_task`, …); semantics are the same. If they are
deferred, load them first (e.g. `ToolSearch select:` for `orchestrator_capabilities`,
`delegate_task`, `task_status`, `task_cancel`). If an initial tool scan does not show them, make one
bounded direct attempt with the known name before concluding they are absent. ACP fallback when the
tools never appear but `T3_ACP_MCP_NODE` is set:

```text
ELECTRON_RUN_AS_NODE=1 "$T3_ACP_MCP_NODE" ${T3_ACP_MCP_ENTRYPOINT:+"$T3_ACP_MCP_ENTRYPOINT"} acp-mcp-call orchestrator_capabilities '{}'
```

Call `orchestrator_capabilities` once and keep the result: it is the live provider/model catalog,
including each model's `options` and whether the provider `canRunChildTask`. If no provider can run
a child task, say so and stop.

The tester never runs in a new worktree and is never a top-level thread: do not use
`t3_thread_launch` / `create_threads` here. A delegated task runs in the parent's checkout, so it
sees uncommitted changes.

## 2. Choose the tester

Default to:

```json
{ "providerInstanceId": "codex", "model": "gpt-5.6-luna" }
```

If the user names an agent or model, map it onto the catalog verbatim, e.g. "Claude Sonnet" →
`{ "providerInstanceId": "claudeAgent", "model": "claude-sonnet-5-5" }`, "codex 5.5 high" →
`{ "providerInstanceId": "codex", "model": "gpt-5.5", "options": { "reasoningEffort": "high" } }`.
Use only ids and option values the catalog lists; if the requested model is missing or its provider
has `constraints`, tell the user and offer the closest listed one.

## 3. Create the tester — exact T3 tool calls

```json
delegate_task {
  "title": "Browser test",
  "role": "test",
  "mode": "async",
  "clientRequestId": "<stable-slug>-browser-test-r1",
  "target": { "providerInstanceId": "codex", "model": "gpt-5.6-luna" },
  "task": "<complete test prompt from section 4>"
}
```

Save the returned `taskId` (and `childThreadId` for reference only). Use a distinct
`clientRequestId` per round, reused only when retrying that same round's call.

Design around how delegated tasks differ from a long-lived terminal agent:

- **No shared history.** The child gets only the `task` prompt. Put every bit of context in it.
- **No continuation of the child.** Each retest is a fresh `delegate_task`; never `t3_thread_send`
  to `childThreadId`. The tester's context and its own preview tabs do not carry over between
  rounds, so a round must hand over the server URL/port and any state the next round needs.
- **No mid-run question channel.** The tester cannot block on the parent; it reports BLOCKED
  scenarios and `Questions for the coordinator`, which you answer in the next round's prompt.

## 4. Build one complete task prompt

Include all known issue/requirement context rather than forcing the tester to infer intent.

The prompt must tell the tester to:

1. Act as a skeptical senior QA engineer testing the changed behavior in a real browser.
2. Stay source-read-only: do not edit, format, stage, commit, reset, stash, or revert files. It may
   inspect code/diffs, run read-only checks, operate the browser, and start a development server.
3. Use only T3's collaborative browser preview tools from the `t3-code` MCP server for browser
   interaction — not Playwright, Chrome automation, curl as a UI substitute, or desktop Computer
   Use. If those tools are not visible at first, make one bounded direct attempt with the known
   name (in Codex code mode, e.g. `tools.mcp__t3_code__preview_status({})`); if they are genuinely
   unavailable, mark every browser scenario BLOCKED and say so instead of substituting another
   browser. The core sequence:
   - `preview_open { "url": "<url>", "reuseExistingTab": false }` → keep the returned `tabId`;
   - for a dev server prefer `preview_navigate { "tabId": "<id>", "target": { "kind":
     "environment-port", "port": <port>, "path": "/..." } }`;
   - `preview_snapshot { "tabId": "<id>" }` to read page state, semantic elements, diagnostics and a
     screenshot before every interaction and after every navigation;
   - `preview_click` / `preview_type` / `preview_press` / `preview_scroll` / `preview_wait_for`
     with role/text Playwright locators and the same `tabId`;
   - `preview_evaluate` only for inspection the snapshot cannot give (console errors captured on
     `window`, `performance.getEntriesByType('resource')` for failed or slow requests, storage);
   - `preview_snapshot { "tabId": "<id>", "save": true }` for evidence screenshots, quoting the
     returned `screenshotPath`.
4. Inspect the actual changes (`git -C <checkout> status`, `git -C <checkout> diff`, or the supplied
   commit range), project instructions, and relevant code. Verify that the supplied summary matches
   the diff. State the `pwd` and `git status` it saw at the top of the report.
5. Derive a compact test charter from the requirement: primary flow, important boundary/negative
   cases, and likely regressions. State assumptions.
6. Reuse an already-running app when safe (the coordinator lists any known URL/port). Otherwise
   discover the documented dev command and start it as a background process from the checkout,
   logging to a file outside the repo. Never stop or repurpose processes it did not start.
7. Always pass the `tabId` it created so testing cannot target the user's or another session's tab.
   Snapshot → interact → re-snapshot, with fresh locators after navigation.
8. Exercise the UI rather than merely inspect DOM/code. Check visible results plus snapshot
   diagnostics, console errors, and relevant failed network requests. Prefer `preview_wait_for` over
   sleeps. If console or network evidence cannot be observed with the available tools, say so rather
   than reporting "no errors".
9. Avoid destructive production actions, purchases, messages, or irreversible external side
   effects. If required credentials, seed data, product decisions, or permission are missing, stop
   that scenario and report exactly what is needed.
10. Report each scenario as PASS, FAIL, or BLOCKED with steps, expected result, actual result, and
    concrete evidence (screenshot paths, diagnostics, evaluated values). For failures include
    severity and the most relevant code area if known. List console/network errors separately,
    distinguish pre-existing noise, give an overall verdict, list `Questions for the coordinator`
    (or `none`), and list everything it created under `Resources created` (preview `tabId`s, dev
    server command + port + log path + PID).
11. Write the full report to `<report file>` — a path in the coordinator's scratchpad, never inside
    the repo — then end the turn with only a compact summary (PASS/FAIL/BLOCKED counts, verdict,
    report path). Do not start more work after that.
12. Before ending: close the preview tabs it created with `t3_preview_close { "tabId": "<id>" }`
    (they belong to its own child thread and cannot be reused by the next round) but leave any dev
    server it started running unless told cleanup is safe, so the next round can reuse it.

Also tell it what is already verified and which server URL, credentials, test tenant, or seed record
is safe to use when known. Never include secrets in the prompt if an existing authenticated session
can be used instead.

## 5. Wait for the notification and read the result

After `delegate_task` returns `status: running`, **end your turn** (or keep doing unrelated work).
Do not poll, sleep, or spawn watchers: T3 wakes this thread when the child reaches a terminal state.
If the user asks meanwhile, say the tester is still running — never predict its results.

When notified (or when you need the result mid-turn):

```json
task_status { "taskId": "<taskId>" }
```

Only `status: completed` with `workState: result_available` settles the round;
`waiting_for_children` is not completion. To stop a tester early:
`task_cancel { "taskId": "<taskId>", "reason": "<why>" }`.

**A `cancelled` or `failed` status is not a verdict.** A child can end `cancelled` without anyone
cancelling it. Read `latestTerminalSummary` / `summary` for what it established, tell the user the
round did not complete, and — with their go-ahead or when obviously wanted — re-run it as a new
`delegate_task` (new `clientRequestId`, e.g. `…-r1b`) whose prompt includes the partial results.

Then read the report **file** the tester wrote; it is the source of truth. Fall back to the
`task_status` `summary` only if the file is missing or empty — the summary can be clipped. Do not
silently return a partial report. A tester that was blocked and reported cleanly still ends
`completed`; read the report for the verdict.

## 6. Verify failures and retest fixes

Do not accept a tester's diagnosis blindly. Reconcile every FAIL against the code and evidence.
Classify it as accepted, refuted with evidence, pre-existing, blocked, or requiring a user decision.
The tester cannot authorize scope expansion or destructive actions.

When the user asked for implementation as well as testing, fix accepted in-scope failures yourself
and run the project's required checks. For every retest, delegate a fresh, self-contained round:

```json
delegate_task {
  "title": "Browser retest <N>",
  "role": "test",
  "mode": "async",
  "clientRequestId": "<stable-slug>-browser-test-r<N>",
  "target": { <same target unless the user changed it> },
  "task": "<original brief from section 4> + previous report (or its failed scenarios with steps and evidence) + answers to its questions + what was fixed, file:line + the running server URL/port from `Resources created`. Retest only the failed and directly affected scenarios against the current working tree. Re-read the diff first. Verify each claimed fix through T3's browser preview, check for new console/network errors, update the verdict, do not edit files, write the full report to <report file>, and end with the compact summary."
}
```

Track each round by its own `taskId`. Repeat until affected scenarios pass, a user decision is
needed, or three rounds have completed. Do not turn a test request into unsolicited code changes
when the user asked only for testing.

Run `git status` before delegating and after each round; if the tester modified files, undo exactly
its edits, re-state the read-only rule, and tell the user.

## 7. Finish and clean up

Summarize:

- tested scenarios and PASS/FAIL/BLOCKED counts;
- reproducible failures and evidence (embed saved screenshots as `![alt](screenshotPath)`);
- fixes made and retest results, if any;
- assumptions, untested risk, the tester model, and the final tester verdict.

In the final round, tell the tester cleanup is safe so it stops only the dev server it started and
closes only the tabs it created, both listed under `Resources created`. If a round is still running
when you finish, `task_cancel` it. Never stop pre-existing servers or close tabs the tester did not
create; report any remaining resources (port, PID, log path) to the user instead of guessing.
