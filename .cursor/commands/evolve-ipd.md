# evolve-ipd

Analyze completed or explicitly accepted work and answer one question:

> Is there anything here that should change how IPD performs future work?

The command reconstructs enough of the delivery experience to identify a probable method-level problem and, when justified, proposes one minimal evolution candidate. It is read-only. It does not turn an observation into canon, create a new workflow object, or modify a closed work item.

**Language:** reply in the same language the user used. Keep paths, IDs, frontmatter keys, and code identifiers in English.

**Lifecycle:** `integration-platform-documentation/00-governance/work-session-lifecycle.md`
**Mode A:** `integration-platform-documentation/00-governance/ipd-program-playbook.md`
**Work item SoT:** `integration-platform-documentation/90-operations/work-items/<work-item-id>/README.md`

---

## When to use

- After delivery is closed or the owner explicitly accepted the result
- When human corrections, rework, missed knowledge, or process failures may reveal a reusable weakness
- When the owner wants to check whether an earlier method change helped on later comparable work

Do not run it automatically after every delivery. Do not use it during open implementation, as a substitute for acceptance or inspection, or to maintain product current-state facts.

---

## Invocation

```text
/evolve-ipd
/evolve-ipd <completed work description or id>
/evolve-ipd <earlier method change and later comparable work>
```

Invocation authorizes read-only analysis and a proposal in chat. It authorizes no file write, work-item creation/update, canon change, product change, commit, or push.

---

## Inputs

Read only what is needed:

1. Completed work item, or explicit owner acceptance when no item exists
2. Relevant transcript and cited prior chats
3. Decisions, handoff, diffs/commits, verification, inspection, and acceptance that shaped the result
4. Current IPD canon and command/workflow contracts implicated by a probable root cause
5. Existing `## Design learning (recommendations)` when present
6. For effect verification, the approved method change and later comparable work
7. `## Method observation` sections in closed method-change work items, to check whether the current work falls under an earlier observation question

Ask only for evidence that is both missing and material. A transcript is evidence, not authority. A final diff alone is not a delivery episode.

---

## Hard rules

1. The useful unit is the **task/delivery episode**, not the whole chat and not only the final diff.
2. Reconstruct only enough of the episode to explain meaningful human interventions, failures, rework, missed knowledge, and consequential decisions. Do not produce exhaustive event logging.
3. Focus on root cause, not on the fact that a correction occurred.
4. Root-cause distinctions are analytical prompts, not enums, schemas, entities, or a required taxonomy.
5. Separate:
   - a local incident or accidental workaround;
   - a human preference;
   - a probable repeatable weakness in the method.
6. A recommendation or evolution candidate is not a rule. Human discussion and decision come before any canon change.
7. Prefer one minimal change to an existing canonical rule, command, or workflow. Do not propose an RFC system, parallel knowledge hierarchy, metrics platform, new agent, or new workflow stage.
8. IPD is durable method memory, not an event store. Conversations, temporary observations, and one-off incidents do not automatically become IPD.
9. Do not duplicate design analysis. If `/analyze-design-work-ipd` already produced recommendations, consume them as evidence and analyze whether any indicate a method-level change.
10. Do not repurpose `/maintain-ipd`. Canon Maintenance aligns verified product current-state facts after push; method evolution is a human-owned semantic decision.
11. Do not claim that the method improved merely because a rule changed. Improvement requires later comparable work and evidence that the motivating problem was reduced.
12. Respect the user constitution: no hidden decisions, no scope widening, and no writes before an exact file/section/change/reason preview plus explicit approval.

---

## Root-cause prompts

Use only the distinctions that help explain the evidence:

- required knowledge did not exist;
- knowledge existed but was not retrieved or hydrated;
- knowledge was available but misunderstood;
- architecture or canon did not define the answer;
- cross-repository or cross-boundary context was missed;
- an API or contract was misunderstood;
- reasoning was poor despite sufficient context;
- validation should have caught the problem;
- the original requirement was genuinely ambiguous;
- the intervention was a human preference rather than a method failure.

Prefer a concrete causal statement:

```text
The ownership rule existed in canon but was not included in the execution context.
```

Do not replace it with a generic label such as “the agent was corrected”.

---

## Execution flow

### Step 1 — Confirm completion

Resolve completed or explicitly accepted work. If delivery is still open, stop. When several completed items are plausible, ask once.

### Step 2 — Reconstruct relevant episodes

Segment only when goal, repository/boundary, approval, runtime, or change set materially changes. For each relevant episode identify:

- intended result and boundaries;
- consequential decisions;
- important human corrections;
- incorrect assumptions and rework;
- failures later corrected;
- knowledge that was missing, missed, or misunderstood;
- accidental workarounds;
- verification and acceptance.

Ignore ordinary implementation detail that does not help explain the method.

### Step 3 — Analyze root cause

For each meaningful intervention or failure:

1. State what happened.
2. Identify the most specific supported root cause.
3. State whether it is local, preference-only, or probably method-level.
4. Cite the evidence and bound negative claims to the inspected corpus.

Patterns across episodes strengthen a method-level conclusion. One episode may still justify a candidate when the causal evidence is strong. Human decision remains mandatory.

### Step 4 — Propose at most one minimal evolution candidate

Only when the evidence supports a probable method problem, answer:

- **What happened?**
- **Why is this probably a method problem rather than a one-off incident?**
- **What evidence supports that?**
- **What is the smallest proposed change?**
- **Where should it live in the existing IPD canon, command, or workflow?**
- **What future work should it affect?**
- **How will later comparable work show whether it helped?**

If no method-level candidate is justified, say so. Do not invent a change to make the run appear useful.

### Step 5 — Discuss and stop

Present the reasoning and candidate in chat. Do not write anything.

If the owner rejects the candidate, no artifact is created. If the owner is undecided, continue discussion without creating a queue.

If the owner approves the candidate, treat the canon change as ordinary Mode A work through `/start-work-ipd`: use the existing work-item lifecycle, show the exact file/section/change/reason plan, wait for implementation approval, then change the existing owning surface. Do not create an `EvolutionCandidate` entity or status.

The method-change work records the motivating evidence, the class of comparable work, and a concrete future observation question under `## Method observation` in its normal body. The heading makes the question findable; it is not a field or a status. Closing that work means **the method changed**, not that the method already improved.

### Step 6 — Observe effect when later evidence exists

On every run, look back: find `## Method observation` sections in closed work items and report whether the current work belongs to the class of comparable work named there and whether the original failure recurred. Report only matches; do not list every observation.

Comparable work is work of the class named in the observation. On that work, inspect only the failure signature the change intended to reduce:

- recurrence of the same human correction;
- correction/rework loop;
- avoidable human attention;
- first-pass acceptance where relevant;
- recurrence of the specific validation failure.

Use only signals that answer the concrete question. No dashboard or universal score is required.

Report one of these conclusions in ordinary prose:

- evidence suggests the change helped — at least one comparable work without recurrence;
- the problem recurred and the change needs reconsideration;
- there is not yet enough comparable evidence.

The sample is small and the conclusion is qualitative. Do not introduce lifecycle states for these conclusions.

---

## Output shape

```markdown
## Experience

**Completed work:** …
**Evidence inspected:** …

### Relevant episodes
- …

### Meaningful interventions and root causes
- **What happened:** …
  **Root cause:** …
  **Assessment:** local | preference-only | probable method problem
  **Evidence:** …

## Evolution

**Candidate:** none | <one minimal change>
**Why method-level:** …
**Owning existing surface:** …
**Future applicability:** …
**Observation question:** …

**Writes:** none
**Human decision required:** yes
```

The assessment words describe the analysis; they are not stored enums.

---

## Related

- Start approved Mode A change: `/start-work-ipd`
- Close delivery: `/close-work-ipd`
- Detailed design learning: `/analyze-design-work-ipd`
- Product current-fact maintenance: `/maintain-ipd`
- Command catalog: `.cursor/commands/README.md`
