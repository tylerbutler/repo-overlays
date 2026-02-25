---
name: ado-issues
description: Query and manage Azure DevOps work items (tasks, bugs, issues, user stories) in the FluidFramework ADO instance. Use when looking up ADO work items, searching for bugs or tasks, checking issue status, or managing work item state. Triggers on mentions of ADO issues, work items, tasks, bugs, or backlog items.
---

# ADO Issues

Query and manage work items from the FluidFramework Azure DevOps instance.

## Configuration

- **Organization**: `fluidframework`
- **Project**: `internal`
- **Base URL**: `https://dev.azure.com/fluidframework/internal`

Always use the `internal` project for work item queries. The `public` project is for pipeline/build queries only.

## Authentication

Ensure `az devops` CLI is configured:

```bash
az devops configure --defaults organization=https://dev.azure.com/fluidframework project=internal
```

## Common Workflows

### Get a Work Item by ID

```bash
az boards work-item show --id <WORK_ITEM_ID> --org https://dev.azure.com/fluidframework --project internal
```

### Search Work Items with WIQL

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.AssignedTo] FROM workitems WHERE [System.TeamProject] = 'internal' AND [System.State] <> 'Closed' AND [System.Title] CONTAINS '<SEARCH_TERM>'" --org https://dev.azure.com/fluidframework --project internal
```

### List Work Items by Type

```bash
# Bugs
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State] FROM workitems WHERE [System.TeamProject] = 'internal' AND [System.WorkItemType] = 'Bug' AND [System.State] <> 'Closed'" --org https://dev.azure.com/fluidframework --project internal

# Tasks
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State] FROM workitems WHERE [System.TeamProject] = 'internal' AND [System.WorkItemType] = 'Task' AND [System.State] <> 'Closed'" --org https://dev.azure.com/fluidframework --project internal
```

### REST API Fallback

```bash
# Get work item by ID
curl -s "https://dev.azure.com/fluidframework/internal/_apis/wit/workitems/<WORK_ITEM_ID>?api-version=7.1" | jq '{id: .id, title: .fields["System.Title"], state: .fields["System.State"], type: .fields["System.WorkItemType"]}'

# WIQL query via REST
curl -s -X POST "https://dev.azure.com/fluidframework/internal/_apis/wit/wiql?api-version=7.1" \
  -H "Content-Type: application/json" \
  -d '{"query": "SELECT [System.Id], [System.Title], [System.State] FROM workitems WHERE [System.TeamProject] = '\''internal'\'' AND [System.State] <> '\''Closed'\''"}'
```

## Tips

- Work item IDs are unique across the entire organization, not just within a project.
- WIQL (Work Item Query Language) uses SQL-like syntax but only supports SELECT and WHERE (no JOINs for flat queries).
- Use `az boards query` for simple lookups; use the REST API for more complex operations.
- Common work item types: `Bug`, `Task`, `User Story`, `Feature`, `Epic`.
- Common states: `New`, `Active`, `Resolved`, `Closed`.
