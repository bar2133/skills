---
name: data-engineering-weekly-report
description: Generates the Data Engineering team weekly/sprint status report from Azure DevOps. Organized by Epic, with objectives, on/off track status, risks, completed and planned items per area, clickable ADO links, and unplanned items flagged. Use when the user asks for a Data Engineering sprint report, weekly report, current sprint status, activities report, or sprint summary.
---

# Data Engineering Weekly / Sprint Status Report

Generates a compact, Epic-based status report for the **Data Engineering** team from Azure DevOps (project **NLASTIC**), then outputs it as markdown. Optionally save to a file or publish to a wiki.

**Data source:** Azure DevOps CLI (`az boards`). Use **az boards iteration team list**, **az boards area team list**, **az boards query** (WIQL), and **az boards work-item show**.

---

## Defaults

| Parameter        | Value                                                        |
| ---------------- | ------------------------------------------------------------ |
| Project          | **NLASTIC**                                                  |
| Team             | **Data Engineering**                                         |
| ADO link format  | `https://dev.azure.com/NCS-RnD/NLASTIC/_workitems/edit/{ID}` |
| Sprint board URL | `https://dev.azure.com/NCS-RnD/NLASTIC/_sprints/taskboard/Data%20Engineering/NLASTIC/{Iteration Name}` |

Use these unless the user specifies otherwise.

---

## Workflow

### 1. Resolve current sprint

- Run `az boards iteration team list --project "NLASTIC" --team "Data Engineering" --timeframe current --output json`.
- From the response take the **iteration path** (and name/dates for the report header).

### 1b. Resolve team area path

- Run `az boards area team list --team "Data Engineering" --project "NLASTIC" --output json`.
- From the response take the **area path** value (e.g. `NLASTIC\Data Engineering`). This is needed to scope the WIQL query to the team's items only.

### 2. Get work items in sprint

- Run a WIQL query filtering by **both** iteration path and area path:
  ```
  az boards query --wiql "SELECT [System.Id] FROM WorkItems WHERE [System.IterationPath] = '<iteration path from step 1>' AND [System.AreaPath] UNDER '<area path from step 1b>'" --project "NLASTIC" --output json
  ```
  **Important:** Without the area path filter the query returns items for _all_ teams in the project.
- Collect all returned work item **IDs**.

### 3. Batch get details

- For each work item ID, run:
  ```
  az boards work-item show --id <ID> --output json
  ```
  **Note:** `az boards work-item show` does not support `--project` or `--fields` flags — it uses the configured default org/project and returns all fields. Extract the needed fields client-side from the `fields` object in the JSON response.
- Fields to extract: `System.Id`, `System.Title`, `System.State`, `System.WorkItemType`, `System.AssignedTo`, `System.Parent`, `System.CreatedDate`, `Microsoft.VSTS.Scheduling.RemainingWork`, `Microsoft.VSTS.Scheduling.TargetDate`, `Microsoft.VSTS.Scheduling.StoryPoints`, `Microsoft.VSTS.Common.ClosedDate`, `Microsoft.VSTS.Common.StateChangeDate`, `Microsoft.VSTS.Common.Severity`.
- Loop over all IDs. Run calls in parallel (e.g. 10 concurrent) to reduce latency.

### 4. Build Epic → Feature → User Story hierarchy

The report is organized **by Epic**, not by team member.

1. From the batch result, separate items by **System.WorkItemType** into **User Story**, **Task**, **Bug**, **Feature**, **Epic**.
2. For each **User Story**, **Bug**, and **Task** in the sprint, read the **System.Parent** field from the details already fetched in step 3.
   - If the parent ID is not already in the sprint results, fetch it: `az boards work-item show --id <parent_id> --output json`.
   - Run these calls in parallel where possible.
3. **Walk the full parent chain upward** — repeat fetching unknown parents until every chain reaches an Epic or a node with no parent. The chain can be 2–4 levels deep (e.g. Task → User Story → Feature → Epic). Fetch each level in a parallel batch before moving to the next.
4. Build the hierarchy: **Epic** → list of **Features** → list of **User Stories/Bugs/Tasks** in the sprint.
5. Items whose parent chain never reaches an Epic go under **"Sprint Management"**. This includes:
   - Items with `System.Parent = null` (true orphans — no parent at all).
   - Items whose chain stops at a Feature or User Story that itself has no parent.
   - Bugs always go under the **"Bugs"** section regardless of parent chain.

### 5. Per-Epic analysis

For each Epic section:

- **Objective**: Derive from the Feature/User Story titles and descriptions — summarize in 1-2 sentences what the team is trying to achieve in this area.
- **Feature effective status**: A Feature's own `System.State` can lag behind its children. Determine effective status from children: if **any** child item (User Story/Task) is not Closed, treat the Feature as open. Only mark a Feature as done when **all** its children in the sprint are Closed.
- **Status indicator**: Assess on/off track based on effective item states (after excluding Removed/Rejected):
  - ✅ **On Track** — majority of items Active or Closed, no overdue items
  - ✅ **Done** — all items in the Epic are Closed
  - ⚠️ **At Risk** — items still New with significant remaining work, or limited progress
- **Exclude** items with State = `Removed` or `Rejected` — drop them entirely from the report (do not count them in totals or list them).
- **Done this week**: Items with State = Closed (use `Microsoft.VSTS.Common.ClosedDate` or `Microsoft.VSTS.Common.StateChangeDate` to confirm recency)
- **In progress**: Items with State = Active or In Review
- **Next week**: Items with State = New that are planned for upcoming work
- **Unplanned items**: Items created during the sprint (compare `System.CreatedDate` to sprint start date). Tag as **Unplanned** in the report.

### 6. Identify risks

- Infer from: items **Overdue** (TargetDate in the past), **New** with large RemainingWork, **Critical/High** severity bugs, unassigned items, buffer capacity consumed, GA timeline pressure.
- Place risks as a single blockquote line under each Epic section (not separate tables).

### 7. Produce the report

Use the [Report template](#report-template) below. Keep it **compact** — use inline bullet lists, not heavy tables. Every work item ID must be a **clickable ADO link**.

### 8. Send / share (optional)

- **Output in chat**: Always include the full report in the response.
- **Save to file**: Default path: `Data_Engineering_Weekly_Report_YYYY-MM-DD.md` in workspace root.
- **Wiki**: If the user wants it in Confluence/Azure Wiki, use the appropriate CLI or API.

---

## Report template

The report must be **compact and scannable**. Use bullet lists for completed/planned items (not tables). Use tables only for bugs (where severity column adds value). Every ID is a clickable link. Assignee shown as first name only in italics.

```markdown
# Data Engineering — Weekly Report | [Week date range]

**Sprint:** [Sprint name link](sprint-board-url) ([Start] – [End]) | [X] of [Y] stories closed | [N] Features closed this week

---

## [Epic Name](epic-link) — Short Label ✅ On Track / ⚠️ At Risk / ✅ Done

**Objective:** [1-2 sentence summary of what we're trying to achieve in this area]

**Done this week:**

- [#ID](link) Feature/US: Title — _Assignee_ ← includes [#child](link), [#child](link)
- [#ID](link) Title (SP:N) — _Assignee_

**In progress:**

- [#ID](link) Title (SP:N, details) — _Assignee_

**Next week:** [#ID](link) Title (SP:N) — _Assignee_ | [#ID](link) Title — _Assignee_

> **Risk:** [One-line risk with linked IDs]. [Mitigation].

---

## Completed Epics ✅

- [Epic Name](epic-link): Short objective — [#F1](link), [#F2](link) closed
- [Epic Name](epic-link): Short objective — [#F1](link) closed

---

## Bugs ⚠️ [N] High-Severity Open

**Done this week:** [#ID](link) title — _Assignee_ | [#ID](link) title — _Assignee_

**In review:** [#ID](link) title (severity) — _Assignee_

**Open — next week:**

| ID          | Title | Sev  | Assignee | Note          |
| ----------- | ----- | ---- | -------- | ------------- |
| [#ID](link) | title | High | name     | **Unplanned** |

> **Risk:** [Description with linked IDs]. [Mitigation].

---

## Sprint Management & GA Readiness

| ID          | Title        | State        |
| ----------- | ------------ | ------------ |
| [#ID](link) | title (SP:N) | State — note |

> **Risk:** [If applicable].

---

_[View sprint board](sprint-board-url)_
```

---

## Formatting rules

- **Clickable links**: Every work item ID must be a markdown link: `[#ID](https://dev.azure.com/NCS-RnD/NLASTIC/_workitems/edit/ID)`
- **Assignee**: First name only, in italics: _Michael_, _Yair_, _Tomer_, _Sari_, _Tzvi_
- **Story points**: Inline as `(SP:N)` — omit if not set
- **Completed Features**: Show Feature as the primary line, child User Stories as `← includes [#ID](link)` on the same line
- **Completed User Stories** (no Feature parent): Show as individual bullet points
- **Next week items**: Combine on a single line separated by `|` to save space
- **Status indicators**: Place on the same line as the Epic heading: `✅ On Track`, `⚠️ At Risk`, `✅ Done`
- **Risks**: Use blockquote (`>`) under each section — one line, not a table
- **Unplanned items**: Bold tag `**Unplanned**` in the Note column for bugs or in-line for stories
- **Epics that are fully done**: When multiple Epics are Done, group them under a single **"Completed Epics"** heading. Show each as one line: Epic name link, short objective, and key Features/items closed (e.g. `- [Epic](link): [objective] — [#F1](link), [#F2](link)`). A single Done Epic can stay as its own section condensed to ~3 lines.
- **No "Type" column** in tables — the context makes it clear
- **No long descriptions** — title is sufficient
- **Buffer items**: Items titled "buffer" (case-insensitive) are capacity placeholders — exclude them from bullet lists and tables. They still count toward story/task totals in the sprint header.
- **Tables only for**: bugs list (severity column needed) and sprint management items

---

## Checklist before delivering

- [ ] Current sprint resolved via **az boards iteration team list**.
- [ ] Team area path resolved via **az boards area team list**.
- [ ] WIQL query scoped by both iteration path **and** area path.
- [ ] All work items in iteration loaded (parallel fetches, batched if >200).
- [ ] Epic → Feature → User Story hierarchy built via parent resolution.
- [ ] Report organized by Epic (not by team member).
- [ ] Each Epic has: objective, on/off track status, done/in-progress/next-week items, risks.
- [ ] Unplanned items identified and tagged.
- [ ] All IDs are clickable ADO links.
- [ ] Report is compact — bullet lists, not heavy tables.
- [ ] Risks include mitigations and are placed as blockquotes under each section.
- [ ] If user asked to "send" or "save", file written or wiki updated.
