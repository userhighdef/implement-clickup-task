# implement-clickup-task

A [Claude Code](https://claude.com/claude-code) skill that takes a ClickUp task from
ticket to pull request.

A ClickUp ticket is rarely complete on its own. The requirements are spread across
subtasks, comment threads, and attached files, and the rest only becomes clear after
reading the code. This skill gathers all of that first, settles the open questions with
you, and only then writes code.

## What it does

1. Asks up front whether you want a commit and a PR at the end, so it never interrupts you mid-review
2. Reads the ClickUp task in full: main task, every subtask, comment threads, attachments, custom fields, dependencies
3. Picks the repo the task belongs to and confirms with you
4. Reads the relevant code
5. Runs a design interview with [`mattpocock-skills`](https://github.com/mattpocock/skills) `grilling` and `domain-modeling`, and will not start coding until you confirm you agree
6. Implements
7. Commits on a `TASK-ID/short-name` branch and opens a PR assigned to you

## Requirements

- A ClickUp MCP server
- The `mattpocock-skills` plugin, for `grilling` and `domain-modeling`
- The `gh` CLI, if you want pull requests

## Install

```bash
git clone https://github.com/userhighdef/implement-clickup-task.git \
  ~/.claude/skills/implement-clickup-task
```

Then open `SKILL.md` and fill in the **Repositories** table in step 2 with your own
projects. Everything else works as written.

## Use

```
/implement-clickup-task PROJ-412
```

A raw ClickUp ID or a task URL works too.

## Notes

Two ClickUp API details this skill works around, in case you are writing something
similar:

- `include_subtasks: true` returns only a shallow summary of each subtask, without
  descriptions or comments. Fetch each subtask on its own.
- `clickup_get_attachments` has no `custom_task_ids` option, so it needs the raw task
  ID, not `PROJ-412`.
