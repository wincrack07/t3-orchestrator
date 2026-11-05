# t3-orchestrator

An agent skill that turns a thread in [T3 Code](https://github.com/pingdotgg/t3code) into a lead orchestrator. It splits the work you just discussed into child threads, gives each one the right provider, model, and effort, and keeps the work going when an account runs out of usage.

## What it does

- **Discovers what you actually have.** It reads your provider instances, models, and effort options from T3 (`orchestrator_capabilities`) instead of assuming a fixed list, and skips instances that are disabled or not signed in.
- **Knows what each instance is for.** It prefers subscriptions over pay-per-token API instances (API only with your OK), and the account that matches the repository's owner.
- **Routes by task.** Hard design and debugging go to the strongest model with high effort, well-specified features to a strong coding model at medium effort, and research and tests to fast models at low effort. Reviews go to a different model family than the author's. It escalates effort first, then model, only after a failed attempt.
- **Learns unfamiliar models.** When a model name does not reveal its tier, it checks the provider's documentation and records a profile so the next run does not repeat the lookup.
- **Handles exhausted usage.** It detects when a child hits a usage limit, records the instance and its reset time in a machine-wide ledger, and stops the limited thread so auto-resume does not duplicate work. Then it continues the task on another instance in the same worktree with a handoff summary. Once the reset time passes, it probes the instance with a small task before using it again.
- **Survives its own limits.** It checkpoints a run file after every step, so a successor orchestrator, or the same thread after a model switch, can pick up where it left off.
- **Stays inside guardrails.** No merges, deploys, data deletion, permission changes, or paid API use without your authorization, and children inherit the same limits.

## Install

With the [skills CLI](https://skills.sh):

```bash
npx skills add wincrack07/t3-orchestrator
```

Or copy this folder into your agent's skills directory, for example `~/.claude/skills/t3-orchestrator` or `~/.codex/skills/t3-orchestrator`.

Then, in a T3 Code thread, say "orchestrate what we discussed" or "use the t3-orchestrator skill".

## Requirements

- T3 Code with the orchestrator MCP tools (`orchestrator_capabilities`, `delegate_task`, `task_status`, `task_cancel`, `t3_thread_launch`, `t3_thread_read`, `t3_thread_wait`). As a fallback, the skill can drive threads with the `t3 thread` CLI.
- At least one provider instance configured in **Settings → Providers**. Several accounts or providers make the failover useful.

## Notes and limitations

- It was written against T3 Code's documentation and CLI as of v0.1.8 (November 2026). T3 moves fast, so the skill checks `t3 --version` and what the tools report, and adapts when something is missing.
- T3's MCP tools do not expose a "limited" state or a reset time directly. The skill detects exhaustion by reading the thread's activity, so new wording from a provider may need a new pattern in `references/usage-limits.md`.
- An orchestrator cannot switch its own thread's provider through tools. Only T3's model picker can, between accounts that share a configuration folder. The skill hands off to a successor thread instead.
- State lives in `~/.local/state/t3-orchestrator/` and never includes credentials.

## Files

- [`SKILL.md`](SKILL.md): the orchestration workflow.
- [`references/model-routing.md`](references/model-routing.md): how to choose instance, model, and effort.
- [`references/usage-limits.md`](references/usage-limits.md): detecting exhausted usage, failover, and recovery.

## License

[MIT](LICENSE)
