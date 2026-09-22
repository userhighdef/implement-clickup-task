---
name: implement-clickup-task
description: Read a ClickUp task in full (task, subtasks, comments, attachments), study the matching repo, grill the user until the design is settled, then implement it and optionally commit and open a PR. Use this whenever the user points at a ClickUp task and wants it built — for example "implement PROJ-412", "do this ClickUp task", "pick up ABC-1234", or when they paste an app.clickup.com task URL and ask for code. Also use it when the user wants a ClickUp ticket analysed properly before any code is written.
---

# Implement a ClickUp task

A ClickUp ticket is rarely complete on its own. The real requirements are spread across
subtasks, comment threads, and attached files, and the rest only becomes clear after
reading the code. This skill gathers all of that first, settles the open questions with
the user, and only then writes code.

The order matters. Every step exists to remove a reason to guess later.

## Setup

Fill in the **Repositories** table in step 2 with your own projects before first use.
Everything else works as written.

Requires the ClickUp MCP server, the `mattpocock-skills` plugin (for `grilling` and
`domain-modeling`), and the `gh` CLI if you want pull requests.

## Step 0 — Ask what happens at the end

Ask this before doing any work, so the user is not interrupted later while reviewing a
design. Use a single `AskUserQuestion` with three options:

- Commit and open a PR
- Commit only
- Neither — stop after implementing

Remember the answer. It decides whether steps 6 and 7 run.

## Step 1 — Read the ClickUp task in full

The input can be a custom task ID (`PROJ-412`), a raw ClickUp ID, or a task URL.
For a custom task ID, pass `custom_task_ids: true` together with `team_id`.

Resolve `team_id` at runtime with `clickup_get_workspaces` rather than hardcoding it.
The workspace ID is the same value `clickup_get_attachments` wants as `workspace_id`,
so you need it anyway.

Capture the raw task ID from the `clickup_get_task_details` response as well.
`clickup_get_attachments` has no `custom_task_ids` option, so its `entity_id` must be
the raw ID — passing `PROJ-412` there fails.

Keep the task `url` from that same response too. The PR description opens with it, and
rebuilding a ClickUp link by hand later is guesswork.

Then read everything:

| What | Tool |
|---|---|
| Main task | `clickup_get_task_details` with `include_markdown_description: true`, `include_subtasks: true` |
| Each subtask | `clickup_get_task_details` **and** `clickup_get_task_comments`, one call per subtask |
| Comments | `clickup_get_task_comments`, then `clickup_get_threaded_comments` for any comment with replies |
| Attachments | `clickup_get_attachments` with `entity_type: "task"`, then download and actually read each file |
| Custom fields | `clickup_get_task_custom_field_values` |
| Related work | `clickup_get_task_dependencies` |

Two things are easy to get wrong here:

`include_subtasks: true` only returns a shallow summary of each subtask. It does not
include their descriptions or comments, which is usually where the detail lives. Fetch
each subtask on its own.

`clickup_get_task_comments` does not return replies. A thread with five replies looks
like one comment. Follow up with `clickup_get_threaded_comments` whenever a comment
reports having them.

Attachments are worth the effort. Screenshots of a bug, a spreadsheet of test cases, or
a Postman collection often define the acceptance criteria more precisely than the
description does.

### Rename the session

As soon as you know the task ID and name, rename the session so it is findable later.
Call `mcp__ccd_session_mgmt__set_session_title` with `session_id: "self"` and a title of
the custom task ID, a space, then a short version of the task name:

```
PROJ-412 Fix timezone offset on export
```

Keep it to about six words. A generic title like "ClickUp task implementation" tells you
nothing once several of these sessions are open at once, which is the whole problem this
solves. The app replaces titles it generated itself without asking, so this is not
interrupting.

Skip it if the tool is not available — it exists only in the Claude Code desktop app.

## Step 2 — Pick the repo

Replace this table with your own projects. If your work always lands in one repo, keep a
single row and skip the confirmation below.

| Path | Remote | Base branch |
|---|---|---|
| `~/work/backend` | `your-org/backend` | `main` |
| `~/work/web` | `your-org/web` | `main` |

Infer the repo from what the task describes, then say which one you picked and why, and
let the user confirm before you continue. Guessing wrong here wastes the whole rest of
the run.

Verify with `git -C <repo> rev-parse --is-inside-work-tree` and stop with a clear message
if it fails. A parent folder holding several checkouts is not itself a repo, and failing
here is much cheaper than failing at commit time.

## Step 3 — Read the code

Find the code the task touches: the entry point, the path a request or job takes through
it, and anything existing that already does part of the job. Look for prior art — a
similar feature elsewhere in the repo usually sets the pattern the new code should follow.

Read enough that your questions in the next step are about decisions, not facts. Anything
you could have looked up yourself is not a good question to spend the user's time on.

## Step 4 — Grill

Invoke two skills in order:

1. `mattpocock-skills:grilling`
2. `mattpocock-skills:domain-modeling`

That pair is exactly what `mattpocock-skills:grill-with-docs` does. Call them directly
rather than calling `grill-with-docs`, because that skill sets
`disable-model-invocation: true` — only the user can invoke it by typing it, a skill
cannot. This is not a workaround for something broken; it is the supported path.

`grilling` runs the interview in rounds and will not let you start coding until the
design tree has no open branches and the user has confirmed you agree. `domain-modeling`
writes what you settled into `CONTEXT.md` and `docs/adr/`, so the reasoning survives
after this session ends.

Do not start implementing before the user confirms shared understanding. That
confirmation is the point of this step.

## Step 5 — Implement

Build what was agreed. Follow the repo's existing patterns and its `CLAUDE.md` or
`AGENTS.md` if present. Run whatever tests or type checks the repo already has, and
report real output rather than assuming it passed.

## Step 6 — Commit

Skip this if the user chose "neither" in step 0.

Run `git status` first. If there are modified files unrelated to this task, name them
and ask before going further — the user may be mid-work on something else. Stage only
what this run created or changed, including the `CONTEXT.md` and ADR files from step 4.
A blanket `git add -A` would sweep unrelated work into the PR.

Run `git fetch origin`, then branch from the base branch on the remote rather than from
whatever is currently checked out. A repo is often left on an earlier feature branch.

Name the branch the same as the PR title:

```
PROJ-412/fix-timezone-offset-on-export
```

Custom task ID, a slash, then a short kebab-case name.

The commit subject is one line and describes the change as a whole:

```
<feat|fix|refactor|chore>: <message>
```

Keep it short. No body, no bullet list — the PR description carries the detail. If your
setup adds trailers such as `Co-Authored-By:`, keep them; a trailer is metadata, not part
of the message length.

Examples:

```
fix: use the report timezone when exporting to CSV
feat: validate recipient address before label creation
refactor: extract rate lookup into its own service
```

## Step 7 — Open the PR

Only if the user chose "commit and open a PR".

```bash
gh pr create --base <base-branch> --assignee @me --title "<title>" --body "<body>"
```

Title: `<customTaskId>/<shortTaskName>` — the same string as the branch name.

Body: start with a link to the ClickUp card on its own line, so anyone reading the PR
can reach the ticket without searching for it. Use the task ID as the link text:

```markdown
[PROJ-412](https://app.clickup.com/t/<raw-task-id>)
```

Then a short summary of what the task asked for and what you did, then the technical
changes as a bullet list. Stay at the level of "what changed and where", not
line-by-line detail — the diff already shows that.

Example body:

```markdown
[PROJ-412](https://app.clickup.com/t/86a1x2y3z)

Exports rendered timestamps in the server timezone, so users outside it saw
rows shifted by a few hours. This uses the report's own timezone instead.

### Technical changes
- Pass the report timezone through `ExportBuilder` instead of reading the server default
- Format timestamps at render time rather than at query time
- Cover a non-UTC report timezone in `export-builder.spec.ts`
```

Report the PR URL when it is created.
