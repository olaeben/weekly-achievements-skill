---
name: weekly-achievements
description: Build a dated weekly record of work or personal-project achievements from Git activity across the user's repositories and, when available, Jira activity. Use for weekly summaries, standups, self-reviews, performance reviews, or requests such as "what did I get done this week?"
---

# Weekly Achievements

Build a concise, evidence-based record of what the user accomplished during a chosen week. Combine Git commits across the user's tracked repositories with Jira activity when an authenticated Jira/Atlassian integration is available. Write the dated summary to disk and show the same summary in chat.

Do not invent impact, outcomes, ownership, or completion. Separate verified activity from unavailable data.

## 1. Determine the week window

Use the current local calendar week, Monday through today, unless the user specifies another week or date range.

For the default window, use the following portable shell pattern. It supports macOS/BSD `date` and GNU `date` rather than assuming one implementation:

```bash
today="$(date +%Y-%m-%d)"
weekday="$(date +%u)"

if date -v-0d +%Y-%m-%d >/dev/null 2>&1; then
  week_start="$(date -v-$((weekday - 1))d +%Y-%m-%d)"
else
  week_start="$(date -d "$today - $((weekday - 1)) days" +%Y-%m-%d)"
fi

week_end="$today"
```

When the user specifies a range, use the requested dates exactly. Derive the ISO week label from the start date. On macOS/BSD use `date -j -f "%Y-%m-%d" "$week_start" +%G-W%V`; on GNU date use `date -d "$week_start" +%G-W%V`. If neither date implementation can derive the label, use `YYYY-MM-DD` from the start date as a safe filename fallback and state that the ISO label was unavailable.

## 2. Maintain the repository list

Use `WEEKLY_ACHIEVEMENTS_STATE_DIR` when set; otherwise use `~/.claude/weekly-achievements`. Keep one absolute repository path per line in `repos.txt`:

```bash
state_dir="${WEEKLY_ACHIEVEMENTS_STATE_DIR:-$HOME/.claude/weekly-achievements}"
repo_file="$state_dir/repos.txt"
mkdir -p "$state_dir"
touch "$repo_file"

repo_root="$(git rev-parse --show-toplevel 2>/dev/null || true)"
if [ -n "$repo_root" ] && ! awk -v p="$repo_root" '$0 == p { found=1 } END { exit found ? 0 : 1 }' "$repo_file"; then
  printf '%s\n' "$repo_root" >> "$repo_file"
fi
```

If the user names other repositories by path, append existing absolute paths that are not already present. Read the file line-by-line so paths containing spaces remain intact. Before scanning, ignore blank lines and paths that no longer exist. Validate each remaining path with `git -C "$repo" rev-parse --is-inside-work-tree`; skip invalid paths without aborting the whole run.

## 3. Collect Git activity

Read the user's Git email with `git config --get user.email`. If no email is configured, report that Git collection was skipped and continue with any available Jira collection; do not silently attribute commits to the wrong person.

For each valid repository, run:

```bash
git -C "$repo" log \
  --author="$git_email" \
  --since="$week_start 00:00:00" \
  --until="$week_end 23:59:59" \
  --pretty=format:'%h|%ad|%s' \
  --date=short
```

Skip repositories where the command fails. Extract Jira-style keys with `[A-Z]{2,}-[0-9]+`. Maintain a map of `ticket key -> commits` and a separate bucket for commits without a ticket key. Treat commit subjects as evidence of activity, not proof of business impact.

## 4. Collect Jira activity when available

Use the Atlassian/Jira MCP integration available in the current environment. Do not assume a particular MCP namespace or tool name; inspect the available Jira tools and use the one that supports JQL search.

Run these queries when Jira is authenticated:

1. Activity query:

   ```text
   (assignee = currentUser() OR worklogAuthor = currentUser()) AND updated >= "WEEK_START" ORDER BY updated DESC
   ```

2. Backfill query for ticket keys found in commits but absent from the activity results:

   ```text
   key in (KEY1, KEY2, ...)
   ```

For each issue, collect only the fields available from the integration: key, summary, status, status category, update timestamp, and any explicit transition evidence. Do not claim that an issue moved to Done during the selected week merely because its current status is Done; require a dated transition or other direct evidence.

If Jira is unauthenticated, unavailable, or errors, continue with a Git-only summary and state plainly that Jira activity was skipped and why. Jira failure must not block the weekly report.

## 5. Synthesize the summary

Group by Jira key first, then place commits without a ticket key under `Other work`.

For each group:

- Write one or two accomplishment-oriented bullets in past tense.
- Use concrete verbs and preserve the wording supported by the evidence.
- Mark an item as completed only when the evidence shows completion, such as a dated Jira transition to Done or an explicitly completed change.
- Label current Jira status separately from historical completion when transition evidence is absent.
- Summarize many related commits by theme instead of listing every commit.
- If activity is genuinely sparse, say so directly rather than padding the report.

Do not include secrets, access tokens, full Jira comments, private credentials, or unnecessary issue descriptions in the output.

## 6. Write the report

Use `ACHIEVEMENTS_DIR` when set; otherwise write to `~/achievements`:

```bash
output_dir="${ACHIEVEMENTS_DIR:-$HOME/achievements}"
mkdir -p "$output_dir"
```

Write or overwrite `$output_dir/<ISO_WEEK>.md` for the selected week. Use this structure:

```markdown
# Week of WEEK_START – WEEK_END (ISO_WEEK)

_Repos scanned: <list>_
_Jira: <included | skipped with reason>_

## Completed
- ...

## In progress
- ...

## Other work
- ...
```

Then update or create `$output_dir/INDEX.md` with one link for the selected week. Re-running the same week must replace the existing report and update its existing index line rather than append a duplicate. Use exact filename matching, not substring matching.

## 7. Present the result

Show the same summary directly in chat so the user does not need to open the file. Mention:

- the date window;
- repositories scanned and skipped;
- whether Jira was included or skipped;
- the report path; and
- any evidence limitations.

Keep the final summary useful even when only Git data is available.
