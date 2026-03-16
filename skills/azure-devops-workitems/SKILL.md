---
name: azure-devops-workitems
description: >
  Explore and manage Azure DevOps work items: Epics, Features, User Stories, and Tasks.
  Provides project overview (tree), get epic, get feature, create work item, and edit work item modes.
  Use when the user wants to list epics, view features, browse work items, create tasks,
  edit user stories, or explore the backlog in Azure DevOps.
---

# Azure DevOps Work Items Explorer

Explore and manage Epics, Features, User Stories, and Tasks in an Azure DevOps project
using `az boards` CLI commands. Five modes: overview, get epic, get feature, create, and edit.

**Announce at start:** "I'm using the azure-devops-workitems skill to work with Azure DevOps work items."

**Prerequisite:** `az` CLI + `azure-devops` extension + active login. If missing, use the
[azure-devops-cli](../azure-devops-cli/SKILL.md) skill to set up first.

---

## Project Resolution (all modes)

Used at the start of every mode to determine the target project:

1. If the user provides a project name, use it (pass `--project <name>`)
2. Else read default: `az devops configure --list | grep project`
3. If no default set, ask the user

---

## Mode 1: Overview (Tree)

Display the full work item hierarchy for a project.

**Steps:**

1. Resolve project
2. Query all Epics:

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.AssignedTo] FROM WorkItems WHERE [System.WorkItemType] = 'Epic' AND [System.TeamProject] = '@project' ORDER BY [System.Id]" --project <project> -o json
```

3. For each Epic, fetch child Features via relations:

```bash
az boards work-item show --id <epic_id> --expand relations -o json
```

Parse `relations` array for entries where `rel == "System.LinkTypes.Hierarchy-Forward"`.
Extract child IDs from the `url` field (last path segment), then fetch each child.
Note: the item `id` is at the top level of the response (not inside `fields`).

4. For each Feature, fetch child User Stories/Tasks the same way.

5. Display as an indented tree:

```
Epic #1234 [Active] - Platform Modernization (Assigned: Alice)
  Feature #1235 [Active] - Migrate to new API (Assigned: Bob)
    Story #1240 [New] - Implement auth endpoint (Assigned: Carol)
    Task #1241 [Active] - Write unit tests (Assigned: Dave)
  Feature #1236 [Resolved] - Update CI pipeline (Assigned: Eve)
```

**Fields per line:** ID, State, Title, Assigned To

---

## Mode 2: Get Epic

Show detailed info for a specific Epic and its child Features.

**Steps:**

1. Resolve project
2. If Epic ID provided, fetch it directly. If not, search by title:

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State] FROM WorkItems WHERE [System.WorkItemType] = 'Epic' AND [System.Title] CONTAINS '<search_term>' AND [System.TeamProject] = '@project'" --project <project> -o json
```

If multiple matches, let the user pick.

3. Fetch the Epic with relations:

```bash
az boards work-item show --id <epic_id> --expand relations -o json
```

4. Display the Epic detail card:
   - ID, Title, State, Assigned To
   - Iteration Path, Area Path, Tags, Priority
   - Description (strip HTML to plain text)

5. List child Features in a table below:
   - ID, Title, State, Assigned To, Iteration Path

---

## Mode 3: Get Feature

Show detailed info for a specific Feature, its child User Stories, and their child Tasks.

**Steps:**

1. Resolve project
2. If Feature ID provided, fetch directly. If not, search by title:

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State] FROM WorkItems WHERE [System.WorkItemType] = 'Feature' AND [System.Title] CONTAINS '<search_term>' AND [System.TeamProject] = '@project'" --project <project> -o json
```

If multiple matches, let the user pick.

3. Fetch the Feature with relations:

```bash
az boards work-item show --id <feature_id> --expand relations -o json
```

4. Display the Feature detail card:
   - ID, Title, State, Assigned To
   - Iteration Path, Area Path, Tags, Priority
   - Description (strip HTML to plain text)

5. Fetch child User Stories via `Hierarchy-Forward` relations.

6. For each User Story, fetch child Tasks via `Hierarchy-Forward` relations.

7. Display as an indented tree below the card:

```
User Story #2001 [Active] - As a user I can login (Assigned: Alice)
  Task #2010 [Active] - Implement login API (Assigned: Bob)
  Task #2011 [New] - Write login tests (Assigned: Carol)
User Story #2002 [New] - As a user I can reset password (Assigned: Dave)
  Task #2020 [New] - Design reset flow (Assigned: Eve)
```

**Fields per line:** ID, Work Item Type, State, Title, Assigned To

---

## Mode 4: Create Work Item

Create a new Epic, Feature, User Story, or Task.

**Steps:**

1. Resolve project
2. Determine work item type (ask if not specified)
3. For non-Epic types, get the parent ID (**mandatory**):
   - Feature -> must have a parent Epic
   - User Story -> must have a parent Feature
   - Task -> must have a parent User Story
   - Do **not** create a Feature, User Story, or Task without a parent
4. Gather fields: Title (required) + optional fields below

### Defaults

| Field          | Default                                                                           |
| -------------- | --------------------------------------------------------------------------------- |
| Assigned To    | Current logged-in user (resolve via `az account show --query "user.name" -o tsv`) |
| Area Path      | `NLASTIC\Data Engineering`                                                        |
| Iteration Path | `NLASTIC` (backlog)                                                               |
| Description    | `TBD` (if user does not provide one — see Description section below for format)   |

### Description

When the user provides a description (or you compose one based on context), write it in **Markdown format**.
Structure the description with headings, bullet points, bold text, and other Markdown elements
as appropriate for the content. This makes descriptions more readable and organized in Azure DevOps.

**Example — User Story description:**

```markdown
## Overview

Allow users to reset their password via email link.

## Acceptance Criteria

- User receives a reset link within 60 seconds
- Link expires after 24 hours
- Password must meet complexity requirements

## Notes

Depends on the email service integration from Feature #1235.
```

**Example — Task description:**

```markdown
## Objective

Write unit tests covering the login API endpoint.

## Scope

- **Happy path:** valid credentials return a JWT token
- **Error cases:** invalid password, locked account, expired session
- **Edge cases:** concurrent login attempts
```

If the user does not provide a description, use the default `TBD`.

### Area Path

Default: `NLASTIC\Data Engineering`.

Before creating, explore sub-Area Paths and suggest a more specific one based on the work item context:

```bash
az boards area project list --project NLASTIC --depth 3 -o json
```

The response is a tree with `path` and `children` fields. Walk the tree to find nodes under
`Data Engineering`.
Suggest the best match based on the work item context. Use the default `NLASTIC\Data Engineering`
if the user accepts or no better match exists.

### Iteration Path

Default: `NLASTIC` (backlog-level, no sprint).

If the user indicates the item is for the **current sprint**, resolve it:

```bash
az boards iteration team list --team "Data Engineering" --project NLASTIC -o json
```

Each entry has `path` and `attributes.startDate`/`attributes.finishDate`. Find the iteration
whose date range contains the current date (match by current month and year). Sprints are
named by month (e.g. `NLASTIC\March 2026`). If no exact match, show available sprints and
let the user pick.

### Confirmation before create

Before executing, display a summary of **all** fields and ask for confirmation:

```
About to create:
  Type:           Feature
  Title:          Migrate to new API
  Assigned To:    bnachlieli@nvidia.com  (from az account)
  Area Path:      NLASTIC\Data Engineering
  Iteration Path: NLASTIC
  Priority:       2
  Parent:         Epic #1234
  Description:    TBD

Proceed? (yes / no / edit fields)
```

**Only execute after the user explicitly confirms.**
If the user says no or wants edits, let them adjust and re-show the summary.

### Create command

**Step 1 — Create the work item (without description):**

```bash
az boards work-item create \
  --type "<Type>" \
  --title "<Title>" \
  --project "<Project>" \
  --fields "System.AssignedTo=<user>" \
           "System.IterationPath=<iteration>" \
           "System.AreaPath=<area>" \
           "System.Tags=<tags>" \
           "Microsoft.VSTS.Common.Priority=<1-4>" \
  -o json
```

Capture the new work item `id` from the JSON response.

**Step 2 — Set the description in Markdown format via REST API:**

The `az boards` CLI stores descriptions as HTML by default. To store the description as rendered
Markdown, use the REST API with `multilineFieldsFormat`:

```bash
az rest --method patch \
  --uri "https://dev.azure.com/<org>/<project>/_apis/wit/workitems/<id>?api-version=7.1-preview.3" \
  --resource "499b84ac-1321-427f-aa17-267ca6975798" \
  --headers "Content-Type=application/json-patch+json" \
  --body '[
    {"op": "add", "path": "/fields/System.Description", "value": "<markdown_description>"},
    {"op": "add", "path": "/multilineFieldsFormat/System.Description", "value": "Markdown"}
  ]'
```

Resolve `<org>` and `<project>` from the configured defaults (`az devops configure --list`).
The `--resource` flag is required for `az rest` to authenticate against Azure DevOps.

**Important:** Once a description is saved in Markdown format, it cannot be reverted to HTML.

### Link to parent (mandatory for non-Epic types)

```bash
az boards work-item relation add \
  --id <new_item_id> \
  --relation-type Parent \
  --target-id <parent_id>
```

**After creation:** display the new item's ID, Title, State, URL, Area Path, Iteration Path,
and parent link confirmation.

---

## Mode 5: Edit Work Item

Update fields on an existing Epic, Feature, User Story, or Task.

**Steps:**

1. Resolve project
2. If item ID provided, fetch it. If not, ask for ID or search by title:

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType] FROM WorkItems WHERE [System.Title] CONTAINS '<search_term>' AND [System.TeamProject] = '@project'" --project <project> -o json
```

3. Display current values for the item
4. Ask which fields to update

### Editable fields

- Title, State, Assigned To, Description
- Iteration Path, Area Path, Tags, Priority

### Confirmation before update

Before executing, display a before/after summary and ask for confirmation:

```
About to update #2001:
  Field           Current                 New
  State           New                     Active
  Priority        3                       2

Proceed? (yes / no / edit fields)
```

**Only execute after the user explicitly confirms.**
If the user says no or wants edits, let them adjust and re-show the summary.

### Update command

For fields **other than Description**, use the standard update command:

```bash
az boards work-item update \
  --id <id> \
  --fields "System.Title=<new_title>" \
           "System.State=<new_state>" \
           "System.AssignedTo=<user>" \
           "Microsoft.VSTS.Common.Priority=<1-4>"
```

**If updating the Description**, use the REST API to set/preserve Markdown format:

```bash
az rest --method patch \
  --uri "https://dev.azure.com/<org>/<project>/_apis/wit/workitems/<id>?api-version=7.1-preview.3" \
  --resource "499b84ac-1321-427f-aa17-267ca6975798" \
  --headers "Content-Type=application/json-patch+json" \
  --body '[
    {"op": "replace", "path": "/fields/System.Description", "value": "<markdown_description>"},
    {"op": "add", "path": "/multilineFieldsFormat/System.Description", "value": "Markdown"}
  ]'
```

Use `"op": "replace"` when the description already has a value, `"op": "add"` for new descriptions.
Both commands can be combined in a single turn if updating description alongside other fields.

**After update:** show a before/after comparison of changed fields and confirm success.

### Common state transitions

| Type           | States                        |
| -------------- | ----------------------------- |
| Epic / Feature | New, Active, Resolved, Closed |
| User Story     | New, Active, Resolved, Closed |
| Task           | New, Active, Closed           |

---

## Quick Reference

| Task             | Command                                                                               |
| ---------------- | ------------------------------------------------------------------------------------- |
| Query work items | `az boards query --wiql "..." --project <project>`                                    |
| Show work item   | `az boards work-item show --id <id> --expand relations`                               |
| Create work item | `az boards work-item create --type <type> --title "<title>" --fields ...`             |
| Update work item | `az boards work-item update --id <id> --fields ...`                                   |
| Link parent      | `az boards work-item relation add --id <id> --relation-type Parent --target-id <pid>` |
| List area paths  | `az boards area team list --team "<team>" --project <project>`                        |
| List iterations  | `az boards iteration team list --team "<team>" --project <project>`                   |
