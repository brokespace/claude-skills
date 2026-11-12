# claude-skills

A Claude Code plugin marketplace (`brokespace-skills`) with two plugins:

| Plugin | Skills |
|---|---|
| `agent-skills` | `agent-system-design`, `agent-tool-design`, `agent-diagnosis` |
| `decision-models` | `decision-model-design` |

Inside a session the skills are namespaced by plugin, e.g.
`/agent-skills:agent-diagnosis` or `/decision-models:decision-model-design`,
and Claude also loads them on its own when a task matches their description.

## Install on a new machine

The repo is public, so no GitHub login is needed.

From a shell:

```bash
claude plugin marketplace add brokespace/claude-skills
claude plugin install agent-skills@brokespace-skills
claude plugin install decision-models@brokespace-skills
```

Or inside a Claude Code session:

```
/plugin marketplace add brokespace/claude-skills
/plugin install agent-skills@brokespace-skills
/plugin install decision-models@brokespace-skills
```

Then turn on auto-update (off by default for non-Anthropic marketplaces):

- In a session: `/plugin` → **Marketplaces** → `brokespace-skills` → **Enable auto-update**, or
- Add `"autoUpdate": true` to the marketplace entry in `~/.claude/settings.json`:

```json
"extraKnownMarketplaces": {
  "brokespace-skills": {
    "source": { "source": "github", "repo": "brokespace/claude-skills" },
    "autoUpdate": true
  }
}
```

Start a new session (or run `/reload-plugins`) to load the skills.

## Updating the skills

Edit the files under `plugins/<plugin>/skills/<skill>/` and push to `main`.

The plugins deliberately have **no `version`** in `plugin.json` or
`marketplace.json`, so Claude Code tracks the git commit: every pushed
commit is a new version. (If you ever add a `version`, you must bump it on
every release or machines will keep their cached copy.)

## How auto-update reaches other machines

With auto-update on, each Claude Code instance refreshes the marketplace in
the background shortly after a session starts and updates the cached plugin
copies under `~/.claude/plugins/cache/`. The running session keeps what it
loaded and shows `Plugin updated … Run /reload-plugins to apply`; the next
session loads the new version automatically.

Auto-update is skipped when `DISABLE_AUTOUPDATER=1`, `DISABLE_UPDATES=1` or
`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` is set, unless
`FORCE_AUTOUPDATE_PLUGINS=1` is also set.

To update by hand:

```bash
claude plugin marketplace update brokespace-skills
claude plugin update agent-skills@brokespace-skills
claude plugin update decision-models@brokespace-skills
```

## Notes

- The skills were written inside a specific codebase, so some passages
  refer to that repo's files, commands and models. The design rules are
  general.
- A few environment details were redacted before publishing (local DB
  credentials, internal repo and client names).
