# Weekly Achievements

A Claude Code skill that turns Git activity across multiple repositories and optional Jira activity into a dated weekly achievements report.

## Install with the Agent Skills CLI

```bash
npx skills add olaeben/weekly-achievements-skill \
  --skill weekly-achievements \
  --agent claude-code \
  --global
```

The same repository can be installed into other supported agents with the `--agent` option. Review the skill before installing it because it can read Git history, use an authenticated Jira integration when available, and write files to the configured output directory.

## Install as a Claude Code plugin marketplace

From Claude Code:

```text
/plugin marketplace add olaeben/weekly-achievements-skill
/plugin install weekly-achievements@weekly-achievements-tools
/reload-plugins
```

The plugin skill is namespaced as `/weekly-achievements:weekly-achievements`.

## Use

Ask Claude Code for a weekly summary, or invoke the skill explicitly:

```text
/weekly-achievements
```

By default it scans the current ISO week, Monday through today. You can request a different week or date range.

## Configuration

- `WEEKLY_ACHIEVEMENTS_STATE_DIR` changes the location of the persistent `repos.txt` list. The default is `~/.claude/weekly-achievements`.
- `ACHIEVEMENTS_DIR` changes the report directory. The default is `~/achievements`.
- Git activity requires a configured `git user.email`.
- Jira activity is optional and requires an authenticated Atlassian/Jira MCP integration.

## Platform support

The date-window logic supports macOS/BSD `date` and GNU `date`. Git repository scanning works with repository paths containing spaces and skips deleted or invalid paths. Jira failures fall back to Git-only reporting.

## License

MIT. See [LICENSE](LICENSE).
