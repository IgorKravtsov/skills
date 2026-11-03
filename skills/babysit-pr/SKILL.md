---
name: babysit-pr
description: >-
  Open a pull request into a target branch, then babysit it through automated review: detect which
  AI/bot reviewers actually started on it (CodeRabbit, SonarQube, Copilot, Claude, Codex, Greptile,
  Cursor Bugbot, …), and for every comment they leave either fix it (commit + push) or push back with
  evidence, always replying in that comment's own thread, until every reviewer is finished and
  satisfied or has broken (rate limit, quota, crash, no response). If no reviewer starts, there is
  nothing to babysit and the skill stops after opening the PR. Use when the user says "babysit the
  PR", "create a PR and watch the reviewers", "handle CodeRabbit comments", "открой PR и следи за
  ревьюерами", "отвечай CodeRabbit", "доведи PR через ревью".
---

# Babysit a pull request through automated review

You own the PR until the bots are done with it. The job is not "open a PR"; it is "open a PR and
leave it with no open bot thread", where every thread ends in either a pushed fix or a reasoned
reply.

Scope: **bot / AI reviewers only**. Human comments are never answered or fixed autonomously:
collect them and hand them to the user.

## 1. Open the PR

1. Follow the repository's own rules first (AGENTS.md / CLAUDE.md / CONTRIBUTING): base branch,
   commit approval, commit message style, wrap-up files. The user asking for a PR counts as approval
   to commit the diff they have already reviewed; if they have not seen the diff yet, show it and ask.
2. Base branch: the one the user named; otherwise the repo's documented default; otherwise ask.
3. Commit only what belongs to the change. Check `git status` for stray files, secrets, `.env`.
4. Push the branch (`git push -u origin <branch>`; never force-push a branch someone else pushed to).
5. `gh pr create --base <base> --title … --body-file …`. The body states the problem, the fix and
   the verification evidence, in the house style of recent merged PRs (`gh pr view <recent> --json body`).
   Never open it as a draft unless asked: most bots skip drafts.
6. In T3 / T4 Code, call `link_pull_request` with the PR URL right after creating it.

## 2. Did any reviewer start?

Reviewers to look for are bots, not people: logins ending in `[bot]` or with `user.type == "Bot"`
(`coderabbitai[bot]`, `sonarqubecloud[bot]`, `copilot-pull-request-reviewer[bot]`,
`claude[bot]`, `chatgpt-codex-connector[bot]`, `greptile-apps[bot]`, `cursor[bot]`, …), plus status
checks / check runs whose name is a reviewer (`CodeRabbit`, `SonarCloud Code Analysis`, …).
Learn the repo's usual set from one recent merged PR before waiting:

```bash
gh api repos/{owner}/{repo}/issues/<n>/comments --jq '.[]|select(.user.type=="Bot")|.user.login' | sort -u
gh api repos/{owner}/{repo}/pulls/<n>/reviews   --jq '.[]|.user.login' | sort -u
ls .coderabbit.yaml .github/copilot* .greptile* 2>/dev/null   # configured but maybe disabled for this base
```

Then give the new PR a bounded window (about 5 minutes, checking every ~60 s) for any sign of
life: a bot comment ("review in progress", walkthrough), a bot review, or a reviewer check that is
queued / in progress / done.

```bash
gh pr checks <pr>
gh api repos/{owner}/{repo}/issues/<pr>/comments --jq '.[]|select(.user.type=="Bot")|{login:.user.login,body:.body[0:200]}'
gh api repos/{owner}/{repo}/pulls/<pr>/reviews  --jq '.[]|{login:.user.login,state,body:.body[0:200]}'
```

- **Nobody started** → report that (and why, if visible: base branch not in the bot config, draft,
  bot not installed) and stop. Do not set up a watch.
- A reviewer **started but is blocked** (e.g. CodeRabbit "Review rate limited" with a wait time) →
  it is alive; wait the stated time, re-request once (`@coderabbitai review` as a PR comment), and
  only if it is still blocked treat it as broken (section 5).

## 3. Watch, do not poll blindly

- **T3 / T4 Code**: `watch_pull_request` with the PR URL, handle everything already on the PR, then
  **end your turn**. The host wakes you on new comments/reviews, failing or passing checks, or a base
  conflict. Do not run `sleep` loops or your own watcher alongside it.
- **Elsewhere**: `ScheduleWakeup` / `Monitor` (or `/loop`) with a delay matched to the bot
  (CodeRabbit usually answers in 2–10 minutes). Keep each wait bounded.

## 4. Each wake-up: triage every new bot item

Gather everything new since the last pass:

```bash
# inline review threads with resolution state (the source of truth for "open")
gh api graphql -f query='query($o:String!,$r:String!,$n:Int!){repository(owner:$o,name:$r){pullRequest(number:$n){
  reviewThreads(first:100){nodes{id isResolved isOutdated path line
    comments(first:50){nodes{databaseId author{login} body createdAt}}}}}}}' -f o=<owner> -f r=<repo> -F n=<pr>
gh api repos/{owner}/{repo}/pulls/<pr>/reviews      # review bodies: CodeRabbit puts "outside diff range" and nitpicks here
gh api repos/{owner}/{repo}/issues/<pr>/comments    # top-level bot comments, summaries
gh pr checks <pr>                                    # quality gates (Sonar etc.) and CI
```

For **each** bot item, read the code it points at and decide on evidence, not on tone:

| Verdict | Action |
| --- | --- |
| **Relevant** (real bug, broken contract, missing case, valid repo-rule violation) | Fix it the way the repo wants (smallest correct change, house style), run the checks that cover it (tests / typecheck / lint / format on touched files), commit with a message naming the finding, push. Reply in the thread: what changed and the commit SHA. |
| **Partly relevant** | Fix the valid part, explain in the thread what was left and why. |
| **Not relevant** (wrong premise, out of scope, contradicts a documented decision or the tech lead's direction, pure style the repo does not want) | Do not change code. Reply in the thread with the concrete reason: file:line, the decision record, the test that proves it. Be short and factual. |
| **Unsure / product decision / needs the user's call** | Do not guess. Reply that it is escalated, and list it for the user. |

Rules for replies:

- Reply **in the item's own thread**, never as a loose PR comment when a thread exists:
  `gh api repos/{owner}/{repo}/pulls/<pr>/comments/<comment_id>/replies -f body='…'`
  (`comment_id` = the thread's first comment `databaseId`).
- Items with no thread (review-body nitpicks, "outside diff range" notes, summary comments): answer
  in one PR comment that mentions the bot (`@coderabbitai …`) and quotes each item it answers.
- Do not resolve threads yourself unless the repo expects it; let the bot resolve after it
  re-checks. Address a bot by its handle when you want it to re-check (`@coderabbitai` …).
- One push per wake-up batch: group the fixes, so each push triggers one incremental review.
- Never answer a human reviewer on the user's behalf; collect their comments for the final report.
- Failing CI or a quality gate on the PR counts as a finding: fix it if the failure is caused by the
  change; if it is pre-existing or infrastructure, say so in the report and do not churn the branch.

After pushing, the bot normally re-reviews by itself (CodeRabbit `auto_incremental_review`). If it
does not within its usual time, request it once (`@coderabbitai review`).

## 5. When to stop

Stop when **every reviewer that started** is in one of these end states:

- **Done and satisfied**: its latest pass on the current head has no unresolved actionable thread,
  and it says so (approved, "LGTM", "No actionable comments", passing quality gate), or every open
  thread carries your reasoned reply and the bot has not pushed back on it.
- **Broken**: rate limited or out of quota after one re-request, errored, its check failed for
  infrastructure reasons, or it has not reacted to the head commit within ~30 minutes after a
  re-request.
- **Deadlocked**: the same point bounced twice with no new argument from either side. Stop arguing,
  escalate it to the user.

Hard cap: 10 fix-and-push rounds. More than that means the review is going in circles; stop and report.

Then:

1. In T3 / T4 Code: `unwatch_pull_request` (the thread goes back to the user's inbox) and
   `list_thread_pull_requests` to confirm the PR is linked.
2. Report to the user, in their language. The report covers:
   - each reviewer and its end state;
   - every finding with its verdict and the commit that fixed it, or the reason it was declined;
   - anything escalated;
   - human comments;
   - checks that are still failing.
3. Never merge, approve, or close the PR yourself unless the user asked.
