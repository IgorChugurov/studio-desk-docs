# start-work-sd

Entry point for any new piece of StudioDesk work. Understand the task, find or create its work item, route it into the feature workflow, and stop. This command does not do research, design, or implementation.

**Language:** this file is English. Reply to the owner in the owner's language, in plain human language. Keep paths, IDs, and code identifiers in English.

**Canon:** `00-governance/work-items.md` (work item format), `00-governance/feature-workflow.md` (stages), `00-governance/approval-rules.md` (approval).

## Invocation

```text
/start-work-sd <description>
```

`<description>` is natural language. Never ask the owner for an ID.

**Authorization:** this invocation authorizes creating and updating the work item README in `90-operations/work-items/` only. It does not authorize writes to any other file, commits, or push. Before any other write, show the file, section, change, and reason, and wait for approval.

## Hard rules

1. Follow the user constitution: no hidden decisions, no scope widening, "OK" approves only what was named.
2. **Understanding gate.** Reply first with the understood task in 2-3 sentences, then stop. No reading of code or documents beyond the work item search, no research, no creating anything until the owner confirms or corrects.
3. If the task has more than one material reading, ask one numbered question and stop.
4. A list of tasks is a sequential queue: one active item at a time. Do not start the next item until the current one is closed or parked.
5. Prefer continuing an unfinished item over creating a duplicate.
6. Anything the owner did not ask for (tasks, hypotheses, improvements) is a labeled proposal, at most one short line at the end, never acted on.
7. After routing, the work continues under `feature-workflow.md`. Do not return to this command after each discussion step. Return only when the item is closed, a new task starts, or the owner asks to replan.

## Flow

### Step 1 - Understand

State the understood task (rule 2) and stop. Wait for the owner.

### Step 2 - Find unfinished items

After confirmation, scan `90-operations/work-items/*/README.md` headers for `in-progress` and `parked`.

| Result | Action |
|---|---|
| One clear match, same outcome | Resume it. Say so. Restore its goal and boundaries into the chat. |
| Several plausible matches | Short numbered list, ask once. |
| Match exists but the task is a new independent result | Treat as new. |
| None | Create a new item. |

### Step 3 - Create the work item

Create `90-operations/work-items/<yyyy-mm-dd>-<slug>/README.md` per `00-governance/work-items.md`: header (`id`, `title`, `status: in-progress`, `updated_at`), Goal, Boundaries, Stage.

### Step 4 - Route

Name the stage in `feature-workflow.md` the task starts at (full route or light route) and who leads. Write it in the Stage section. Tell the owner in plain language what happens next. Stop.

## Parking

If work stops mid-way, set `status: parked` and write what is done and where to continue.

## Output

```markdown
## Work started

**Item:** <title> (`<id>`)
**Goal:** …
**Boundaries:** …
**Stage:** …
**Next:** …
```
