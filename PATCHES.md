# internal patches

This branch (`internal/main`) tracks `ZSeven-W/dsh-crew` upstream plus the
changes below. Rebase on upstream when it moves; re-apply these on top.

Upstream release this branch is based on: `0.1.0-rc.14`.
Verified against DSH `0.2.2-rc.1` in hub mode.

## 1. Reasoning effort reaches the agent through `agentOptions`

Upstream sends `{ provider, model }` to `agents.create` and re-applies
`reasoningEffort` from the `agent/request` hook. DSH >= 0.2.2 fixes the effort
during *step preparation*, so preparation lands on the adapter default and the
later request disagrees, and the host rejects the turn:

```
agent/request proposed reasoning effort "max" after step preparation
fixed "adapter default"; use agent.model.select() before step preparation
```

Fix: also pass `reasoningEffort` in `agentOptions`, before preparation.

Both sites must carry the effort. Measured on `0.2.2-rc.1`:

| `agentOptions` | hook | result |
|---|---|---|
| effort | effort | **done** |
| — | effort | rejected (prep fixed "adapter default") |
| effort | — | rejected (request proposed "adapter default") |

Submitted upstream as PR ZSeven-W/dsh-crew#21. Drop this patch once that lands.

## 2. `dsh_run_worker` is non-blocking by default

The call returns `started subagent <childId>` as soon as the runtime accepts the
child, matching the official `@deepseek-ai/dsh-tool-subagent` contract, and the
result is read later with `dsh_worker_result`.

A new optional `wait_seconds` keeps the old blocking behaviour when a caller
genuinely cannot continue without the answer. Escalation-to-pro is skipped on
the non-blocking path: there is no result yet to escalate on.

Claude Code has no completion-notice channel, so "end your turn and wait for
the notice" degrades into "end your turn, read the result next turn".

## 3. Dispatcher agents removed

`agents/ds-flash.md` and `agents/ds-pro.md` are deleted. They were a haiku
"thin dispatcher" layer that re-read the task before forwarding it to
`dsh_run_worker`, adding a model hop the official contract does not have.

Dispatch from the main session instead:

```
dsh_run_worker({ task, tier: "pro", cwd: "/abs/path" })
dsh_worker_result({ job_id, wait_seconds: 60 })
```

Restore them from `upstream/main` if the subagent names are wanted back.

## Local notes

- `model: haiku` in the deleted agents was only a placeholder for the
  dispatcher; the real model is chosen by the DSH tier binding.
- Provider bindings live in `~/.config/dsh-crew/config.json`
  (`flash_provider` / `pro_provider`). Under account auth only
  `deepseek-account` has credentials; `deepseek-official` needs an API key.
