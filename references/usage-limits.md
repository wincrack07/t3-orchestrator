# Exhausted usage: detect, switch providers, and come back

Goal: never assign work to an instance with no quota left, keep a task going on another instance without losing progress, and use the instance again once its quota recovers.

## State files

They live outside the repository so git stays clean:

- **Provider ledger, shared by every run on this machine** (quota belongs to the account, not the project): `~/.local/state/t3-orchestrator/providers.json`

  ```json
  {
    "updatedAt": "2026-11-05T15:04:00-05:00",
    "instances": {
      "<providerInstanceId>": {
        "status": "available | limited | probing | broken",
        "window": "5h | weekly | daily | unknown",
        "limitedAt": "ISO-8601",
        "resetsAt": "ISO-8601 or null",
        "probeAfter": "ISO-8601 or null",
        "failures": 0,
        "source": "thread <id>: <short quote of the notice>"
      }
    }
  }
  ```

- **Run state:** `~/.local/state/t3-orchestrator/runs/<repo>-<YYYYMMDD-HHMM>.md`. It holds the scope, the plan, a task table (taskId, childThreadId, instance, model, effort, worktree, branch, status), the decisions, and the next step. Update it after every change. It is what lets another model take over.

Before writing, read the file again. Write to a temporary file and rename it, so you do not overwrite another run. Never store tokens or anything from `secrets/` there.

## Detect exhaustion in a child

Check on every cycle, and whenever a child moves to `failed`, `interrupted`, or `waiting` with no pending question:

1. Read `t3_thread_read` with `view: "activity"` and the latest items.
2. Look, case-insensitively, for notices such as `usage limit`, `rate limit`, `limit reached`, `hit your limit`, `out of usage`, `quota`, `exceeded`, `429`, `insufficient credits`, `resets at`, `try again at`. In T3's UI this state is called **Limited**.
3. Extract the window (5 h, weekly, or daily) and the reset time, if they appear.
4. If the error is about authentication, the network, or the binary rather than quota, mark the instance `broken` and tell the user. That is not exhaustion.

Two more signals:

- **Silent fallback:** `delegate_task` returns the `providerInstanceId` and `model` actually used. If they are not the ones you asked for, T3 rerouted the task, usually because the other instance had no quota. Record the real instance and mark the requested one `limited`, with `resetsAt` null.
- **Shared account:** if a child on your own account runs out, you are next. Go to "If you are the one running out".

## Switch providers without losing work

1. **Record** the instance in the ledger with `status: "limited"` and `resetsAt`, or `probeAfter` if there is no time. The wait grows 1 h, 2 h, 4 h, and 24 h for a weekly window.
2. **Stop the exhausted thread.** If T3's "Resume after usage limits" setting is on, a limited thread continues by itself when the quota resets and would duplicate the work. Call `task_cancel` with the reason, and confirm with `task_status` that it is `cancelled`. Exception: if no other instance can take the task, let it resume by itself and tell the user when.
3. **Salvage the progress.** From `t3_thread_read`, take `thread.worktreePath` and `branch`. Run `git -C <worktreePath> status`, `diff --stat`, and `log` to see what is done.
4. **Choose the replacement** with the same role and the same [routing](model-routing.md), excluding limited instances. First another account of the same driver (it behaves the same), then another driver with `canRunCrossProviderChildTask`. API only with the user's permission.
5. **Continue in the same worktree:**
   - If there is progress, use `t3_thread_launch` with `workspaceStrategy: "existing_worktree"`, the `worktreePath`, a `modelSelection` with the new instance and model, `delegatedFrom` if available, and a handoff `message`. The handoff has the original brief, what is already done (commits and files), what is left, and this instruction: "Continue in this worktree. Do not redo finished work. Escalate questions to the orchestrator."
   - If there was no progress, call `delegate_task` again with the new `target` and a new `clientRequestId`.
6. **Update** the run file and tell the user in one line, for example: "Codex · Personal ran out (5 h, back at 18:40). I moved T3 to Claude · Personal, in the same worktree."

## If you are the one running out

No MCP tool changes your own thread's provider: `t3_thread_update` only renames. Within one thread, the switch can only be made from T3's model picker, and only between accounts that share the same configuration folder (`CODEX_HOME` for Codex, the configuration directory for Claude). So:

- **Before you run out:** save the run state after every step. If you see signs that your quota is running low (a child on your own account already ran out, or a usage warning), write the handoff in the run file. Then launch a successor orchestrator with `t3_thread_launch` on an available instance, with `workspaceStrategy: "root"` and this message: "Resume the run at `<run file path>` with the t3-orchestrator skill". Give the user the link and stop delegating from this thread.
- **If you were resumed** (by auto-resume, or because the user changed the thread's model): before anything else, read the run file and the ledger, and call `task_status` on every open task.

## Use a recovered instance again

At the start of each cycle:

1. Call `orchestrator_capabilities` again, because `constraints` change.
2. If a `limited` instance has a `resetsAt` or `probeAfter` in the past, move it to `probing`.
3. A `probing` instance gets only the next small, low-risk task, as a test:
   - If it finishes with no quota notice, the instance becomes `available` and `failures` goes back to 0.
   - If it runs out again, it goes back to `limited` and the wait doubles.
4. Do not mark it `available` just because the time has passed: the probe is what confirms it.
