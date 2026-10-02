# close-work-sd

Close a finished work item: check that the result exists and how it was checked, write the closure, and commit. This command does not edit canon, does not start new work, and does not push.

**Language:** this file is English. Reply to the owner in the owner's language, in plain human language. Keep paths, IDs, and code identifiers in English.

**Canon:** `00-governance/work-items.md` (work item format and statuses), `00-governance/approval-rules.md`.

## Invocation

```text
/close-work-sd
/close-work-sd --cancel
/close-work-sd <description or id>
```

Infer exactly one work item from context. If several are plausible, ask once.

**Authorization:** this invocation is the owner's explicit request to commit. It authorizes: writing the closure into the work item README, and scoped commits of this item's files. It does not authorize push, edits to other files, or work on another item.

## Hard rules

1. Follow the user constitution: no hidden decisions, no scope widening.
2. Never close incomplete work. If something is missing, report it, leave the status as it is, and stop.
3. Never cancel silently. Cancel only with `--cancel` and a written reason.
4. Closing needs one verification artifact: a screenshot, command or test output, or a quoted owner acceptance line. A claim of "done" without an artifact blocks the close.
5. Commit only this item's files, each repository in its own commit. Do not include files from other tasks or secrets. If the file list is unclear, ask once.
6. The work item record goes into the same commit as the work. The commit message always contains the item id in square brackets. Hashes are not written into the record: the commit is found with `git log --grep "<id>"`.
7. Never push. Push only on a separate explicit owner request.

## Flow

### Step 1 - Find the item

Infer the item. If unclear, ask once.

### Step 2 - Cancel path (only with `--cancel`)

Ask for the reason if missing. Write it in the item, set `status: cancelled`, update `updated_at`, stop.

### Step 3 - Check

Check and report in plain language:
1. The result matches the Goal and Boundaries in the item, or the owner accepted a reduced scope.
2. A verification artifact exists (rule 4).
3. Git state in the affected repositories: no unexpected or unfinished files.

If anything is missing, show the gaps and stop (rule 2).

### Step 4 - Close and commit

1. Write the Closure section: result, verification artifact, and the line "Commits: find with `git log --grep "<id>"`".
2. Set `status: done`, update `updated_at`.
3. Commit the item's files per repository (rules 5 and 6). One commit per repository, nothing after it.
4. Tell the owner what was committed, including the hashes. Push is not done.

## Output

```markdown
## Closed

**Item:** <title> (`<id>`)
**Status:** done
**Result:** …
**Verification:** <artifact>
**Commits:** <repo: hash>
**Push:** not performed
```

If blocked:

```markdown
## Close blocked

**Item:** <title> (`<id>`)
**Gaps:** …
**Status left as:** <status>
```
