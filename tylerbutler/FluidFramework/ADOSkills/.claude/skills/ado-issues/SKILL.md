---
name: ado-issues
description: Look up work items (issues, bugs, tasks, user stories) in Azure DevOps
---

# ADO Work Items Lookup

Use the `az boards` CLI to look up work items in Azure DevOps.

## Configuration

- **Organization**: `https://dev.azure.com/fluidframework`
- **Project**: `internal`

Always use these values with `--org` and `--project` flags.

## Prerequisites

The `azure-devops` extension must be installed:
```bash
az extension add --name azure-devops
```

## Common Commands

### Get a specific work item by ID

```bash
az boards work-item show --id <WORK_ITEM_ID> --org https://dev.azure.com/fluidframework --project internal
```

### Search work items with a query

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State] FROM WorkItems WHERE [System.AssignedTo] = @Me AND [System.State] <> 'Closed'" --org https://dev.azure.com/fluidframework --project internal
```

### List work items assigned to current user

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType] FROM WorkItems WHERE [System.AssignedTo] = @Me ORDER BY [System.ChangedDate] DESC" --org https://dev.azure.com/fluidframework --project internal
```

### List recent work items in a specific state

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.AssignedTo] FROM WorkItems WHERE [System.State] = 'Active' ORDER BY [System.ChangedDate] DESC" --org https://dev.azure.com/fluidframework --project internal --output table
```

### Get work item with all fields

```bash
az boards work-item show --id <WORK_ITEM_ID> --org https://dev.azure.com/fluidframework --project internal --expand all
```

## Output Formatting

Use `--output table` for readable summaries or `--output json` for detailed data.

## Workflow

1. Ask the user for the work item ID or search criteria if not provided
2. Run the appropriate command using the internal project
3. Summarize the results for the user, including:
   - Work item ID and title
   - State and assigned to
   - Description (if relevant)
   - Link to the work item in ADO
