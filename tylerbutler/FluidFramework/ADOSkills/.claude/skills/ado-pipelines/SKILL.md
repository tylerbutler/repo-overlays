---
name: ado-pipelines
description: Check CI pipeline status in Azure DevOps
---

# ADO Pipeline Status

Use the `az pipelines` CLI to check CI pipeline status in Azure DevOps.

## Configuration

- **Organization**: `https://dev.azure.com/fluidframework`
- **Project**: `public`

Always use these values with `--org` and `--project` flags.

## Prerequisites

The `azure-devops` extension must be installed:
```bash
az extension add --name azure-devops
```

## Common Commands

### List all pipelines

```bash
az pipelines list --org https://dev.azure.com/fluidframework --project public --output table
```

### List recent pipeline runs

```bash
az pipelines runs list --org https://dev.azure.com/fluidframework --project public --output table --top 10
```

### List runs for a specific pipeline

```bash
az pipelines runs list --pipeline-ids <PIPELINE_ID> --org https://dev.azure.com/fluidframework --project public --output table --top 10
```

### Get details of a specific run

```bash
az pipelines runs show --id <RUN_ID> --org https://dev.azure.com/fluidframework --project public
```

### List runs for a specific branch

```bash
az pipelines runs list --branch <BRANCH_NAME> --org https://dev.azure.com/fluidframework --project public --output table --top 10
```

### Check pipeline run status with result filtering

```bash
az pipelines runs list --org https://dev.azure.com/fluidframework --project public --status completed --result failed --top 10 --output table
```

### Get build logs

```bash
az pipelines runs show --id <RUN_ID> --org https://dev.azure.com/fluidframework --project public --open
```

## Status Values

- **status**: `notStarted`, `inProgress`, `completed`, `canceling`, `postponed`, `notSet`
- **result**: `succeeded`, `failed`, `canceled`, `partiallySucceeded`

## Output Formatting

Use `--output table` for readable summaries or `--output json` for detailed data.

## Workflow

1. Ask the user what they want to check (specific run, branch status, recent failures, etc.)
2. Run the appropriate command using the public project
3. Summarize the results, highlighting:
   - Run status and result
   - Duration if completed
   - Link to the run in ADO
   - Any failure information

## Tips

- Use `--open` flag to open the run in a browser
- For PR builds, filter by the PR branch name (e.g., `refs/pull/<PR_NUMBER>/merge`)
