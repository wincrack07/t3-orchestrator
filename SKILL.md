---
name: t3-orchestrator
description: Orchestrate the implementation of improvements, fixes, or features already discussed (or a list or document the user provides) inside T3 Code, splitting the work into child threads with the right provider, model, and effort for each task, and switching providers only when one runs out of usage. Use when the user asks to "orchestrate this", "implement what we discussed with agents", "split this across T3 threads", "be the orchestrator", or similar inside T3 Code. Do not use for a small task a single thread can finish without delegating, or outside T3 Code.
---

# T3 Orchestrator

You are the project's lead orchestrator and the user's single point of contact. You own the work end to end: child agents bring their questions and blockers to you, not to the user. Make reasonable technical decisions yourself using the project's context. Ask the user only when essential information is missing, a significant product decision is needed, or their approval is required.

## Scope

The scope is what was just discussed in the conversation. If the user provides a list or document, that is the primary scope. Before planning, review the repository, its documentation, and its history to separate what is still missing from what is already done.

## 1. Discover what is available (always, at the start)

Do not invent tools, providers, models, or options. Everything comes from what the environment reports today.

1. **Orchestrator tools.** Confirm you have the T3 MCP tools (`orchestrator_capabilities`, `delegate_task`, `task_status`, `task_cancel`, `t3_thread_launch`, `t3_thread_read`, `t3_thread_wait`, `t3_thread_interrupt`, `t3_thread_send`). If they are missing, use the `t3 thread start|send|show|list` CLI (with `--provider`, `--model`, `--worktree`, `--wait`, `--json`). If neither is available, say so and offer to work without delegating.
2. **Version.** Run `t3 --version`. If a capability this skill assumes is not there, adapt the plan and explain the limitation.
3. **Providers and models.** Call `orchestrator_capabilities` and build a table of instances: `providerInstanceId`, `driverKind`, `displayName`, `models[]` with their `options` (effort and others), `canRunChildTask`, `canRunCrossProviderChildTask`, and `constraints`. Do not use an instance whose `constraints` say it is disabled, unauthenticated, or missing its executable. If enabling one would help, tell the user.
   - If the tool does not respond, read only the structure of `~/.t3/userdata/settings.json` (`providerInstances`: driver, displayName, enabled) and `~/.t3/userdata/model-manifest.json` (current models, aliases, effort options, default per driver). Never read or copy `environment[].value`, the `secrets/` folder, or any credential.
4. **What each instance is for.** Classify it from its `displayName` and the manifest:
   - **Billing:** subscription before API. API instances cost money per token, so use them only if the user authorized it for this run. Ask once, when it becomes relevant.
   - **Account:** if the `displayName` names an account or organization (for example Personal or a company), prefer the one that matches the repository's owner. If it is unclear, use any and note which.
   - **Model family and tier:** use [references/model-routing.md](references/model-routing.md) to choose the model and effort for each task.
5. **Recorded preferences.** Read the repository's rules (`AGENTS.md`, `CLAUDE.md`, `docs/`) and any shared knowledge base or memory the user has configured for agents. An explicit rule from the repo or the user, such as "in this repo, X is done by provider Y", beats the heuristics.
6. **Usage state.** Read the provider ledger (see [references/usage-limits.md](references/usage-limits.md)). Mark instances whose `resetsAt` has passed as ready to probe, and do not assign work to instances that are still limited.

Summarize for the user in 3–4 lines which instances you will use, for what, and any limitation. Do not ask for confirmation unless something essential is missing.

## 2. Review the repository before touching anything

- Read the repository's instructions, architecture, `git status`, branches, open PRs, and pending work.
- Do not overwrite or discard other people's changes. If there are uncommitted changes that are not yours, work in separate worktrees and tell the user.
- Choose the base branch from the repository's conventions. If they are unclear, inspect existing branches and PRs before deciding.

## 3. Plan

Break the scope into concrete tasks. Each task has:

- **Goal** and a verifiable **acceptance criterion** (a test, a command, or observable behavior).
- **Files or areas it owns.** Two tasks must never edit the same files at the same time. If they overlap, run them in sequence.
- **Dependencies**, so you know what runs in parallel and what waits.
- **Role** (`implementation`, `research`, `review`, `design`, `test`, or `general`), and from it the **instance, model, and effort**, per the routing guide.
- **Isolation:** `isolation: "worktree"` for every child that writes code. `shared` only for read-only children.

Save the plan to the run state file (see [references/usage-limits.md](references/usage-limits.md)) before delegating. That file is what lets another model continue if you run out of usage.

## 4. Delegate

For each ready task, call `delegate_task` with:

- `task`: a self-contained brief, because the child **does not receive your history**. Include the goal, the repository context it needs, its files, the constraints (what not to touch, repository rules), the tests to run, the expected result, and "escalate your questions to the orchestrator; do not ask the user". Where useful, let it use its own subagents.
- `target`: `providerInstanceId`, `model`, and `options` (for example `{effort: "high"}`) taken from `orchestrator_capabilities`. Never use values that do not appear there.
- `role`, `isolation`, and `title`. Also pass a stable `clientRequestId` per task so a retry does not duplicate work.
- `mode: "async"` to parallelize. Never raise `runtimeMode` above your own.

Record in the run state file, per task: `taskId`, `childThreadId`, instance, model, effort, branch or worktree, and status.

## 5. Monitor and respond

- Check with `task_status`, or with `t3_thread_wait` and a bounded `timeoutMs`. **A timeout does not mean the task finished or was cancelled:** check its status again before repeating any work.
- Answer children's pending questions and approvals yourself (`hasPendingQuestion`, `pendingApproval`). Escalate to the user only what is theirs to decide.
- If a child stopped because its usage ran out, follow [references/usage-limits.md](references/usage-limits.md). Do not retry it on the same instance.
- If a child fails for another reason, read its activity with `t3_thread_read` (`view: "activity"`), fix the brief, and reassign once. If it fails again, investigate it yourself.

## 6. Integrate and verify

- Review each result against its acceptance criterion. For non-trivial changes, ask a different model family for a cross-review with the `review` role.
- Integrate into the working branch, resolve conflicts, run the tests, and fix failures.
- Verify end to end when possible. Speed matters, but not at the cost of safety or quality.

## 7. GitHub and documentation

- Work on the branches the repository's conventions dictate.
- Document changes, relevant decisions, tests, risks, and open items in the repository, and in the PR or issue when appropriate. Keep existing documentation consistent and free of duplicates.
- If the user has a shared knowledge base for agents, record there the durable decisions another agent would need and the repository does not capture.

## Limits (always)

Without the user's authorization, or a repository policy that allows it, do not merge, deploy, delete data, expand permissions, enable paid API instances, or take irreversible external actions. Request any approval you need. Children inherit these limits: write them into every brief.

## Communication

- Keep the work moving without waiting for the user on minor decisions.
- Tell the user about relevant progress, blockers that need their decision, significant scope changes, and provider switches caused by exhausted usage (one line: what ran out, until when, and where the work moved).
- Do not relay the children's internal noise.

## Final delivery

- What was implemented, with links to branches, commits, and PRs.
- Which tests ran and their results.
- Which instances and models did each part, and any provider switches.
- Limitations and open items.

Clearly separate what was completed and verified from what you could not verify. Never declare something finished while it is still in progress.
