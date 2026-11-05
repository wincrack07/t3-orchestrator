# Routing: which instance, model, and effort to use

Models change often. This file gives a method for choosing, not a fixed list. Concrete names always come from `orchestrator_capabilities` at the moment you use them.

## Decision order

1. **An explicit rule** from the user or the repository (`AGENTS.md`, `CLAUDE.md`, or the user's knowledge base). It always wins.
2. **Availability:** only instances with no blocking `constraints` that the ledger marks `available` or `probing`.
3. **Billing:** subscription before API. API only if the user authorized it for this run.
4. **The task's role** → model tier and effort, per the table below.
5. **Load spreading:** do not put every task on one account. Keep the strongest instance for hard tasks and send cheap tasks to the one with the most headroom. A child on your own account spends from your own quota.

## Role → tier and effort

| Role | Examples | Model tier | Starting effort |
|---|---|---|---|
| `design` | architecture, data migrations, security, decisions with many dependencies | the most capable available | high, or xhigh if critical |
| hard `implementation` | deep debugging, concurrency, changes across several modules | the most capable, or a strong coding model | high |
| clear `implementation` | a well-specified feature with bounded files | a strong coding model, mid-to-high tier | medium |
| `review` | PR or diff review | a capable model from a **different family** than the author's | high |
| `research` | reading the codebase or docs, mapping, searching | a fast model with a large context | low or medium |
| `test` | writing or running tests, mechanical fixes | a fast or mid-tier model | low or medium |
| `general` | docs, renames, trivial changes | the fastest or cheapest | low |

**Escalation:** start with the least that meets the acceptance criterion. If an attempt fails on quality (not on usage), raise the effort one step first, then the model tier. Record in the run state file why you escalated.

## How to tell a model's tier

`orchestrator_capabilities` does not say what each model is good at. Work it out in this order, and record your source:

1. **What is already recorded:** look in `~/.local/state/t3-orchestrator/model-profiles.json` (or the user's knowledge base, if they keep model notes there) for an entry for that model id. If it was verified less than 30 days ago, use it.
2. **The local manifest** (`~/.t3/userdata/model-manifest.json`): `defaults.chat` is the provider's recommended default. `currentModels` are the current ones. `profiles[*].capabilities.optionDescriptors` lists the effort values each model accepts.
3. **Name signals**, only as a hint:
   - `opus`, `pro`, or `max` usually mean the top tier.
   - `sonnet` usually means the middle tier.
   - `haiku`, `flash`, `mini`, or `lite` usually mean the fast tier.
   - A `-high`, `-medium`, or `-low` suffix in the slug already fixes the effort.
4. **The provider's official documentation**, when the name is not enough (for example, codenames). Do not guess their tier.
5. Once confirmed, add or update the entry in `model-profiles.json`, with the strengths, tier, accepted efforts, source URL, and verification date, so the next run does not repeat the lookup.

If the tier is still unclear and the task is critical, use the provider's default with high effort. Otherwise, use that family's middle model.

## Effort rules

- More effort means better quality on hard problems, but also more latency and more quota spent. Do not raise it by default.
- Do not use modes that start their own multi-agent workflow (they collide with this orchestration), or `max`-style efforts, except for a task that is stuck after a normal attempt, or when the user asks.
- Do not turn on fast modes unless the user asks.
- Use only the values that `options` lists for that model. If the model has no `effort` option, leave it out.

## Cross-provider

- Delegating to a `driverKind` different from yours requires `canRunCrossProviderChildTask = true` on that instance.
- For `review`, prefer a different driver from the one that wrote the code. If there is only one, use another model of that family with higher effort.
