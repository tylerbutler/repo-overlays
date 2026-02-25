---
name: ado-pipelines
description: Query Azure DevOps pipeline results for PRs and builds in the FluidFramework ADO instance. Use when checking CI build status, investigating pipeline failures, retrieving build logs, or diagnosing PR check failures. Triggers on mentions of ADO pipelines, CI failures, build status, PR checks, or pipeline logs.
---

# ADO Pipelines

Query pipeline results and build logs from the FluidFramework Azure DevOps instance.

## Configuration

- **Organization**: `fluidframework`
- **Project**: `public`
- **Base URL**: `https://dev.azure.com/fluidframework/public`

Always use the `public` project for pipeline queries. The `internal` project is for work items/issues only.

## Authentication

Ensure `az devops` CLI is configured:

```bash
az devops configure --defaults organization=https://dev.azure.com/fluidframework project=public
```

## Common Workflows

### Find PR Build Status

Given a PR number or branch name, find the associated pipeline runs:

```bash
# List pipeline runs for a specific branch
az pipelines runs list --branch "refs/pull/<PR_NUMBER>/merge" --org https://dev.azure.com/fluidframework --project public --output table

# Or by branch name
az pipelines runs list --branch "refs/heads/<BRANCH_NAME>" --org https://dev.azure.com/fluidframework --project public --output table
```

If the CLI doesn't return results, use the REST API:

```bash
# List builds for a PR
curl -s "https://dev.azure.com/fluidframework/public/_apis/build/builds?reasonFilter=pullRequest&repositoryId=<REPO_ID>&repositoryType=GitHub&branchName=refs/pull/<PR_NUMBER>/merge&api-version=7.1" | jq '.value[] | {id, buildNumber, status, result, definition: .definition.name, startTime, finishTime}'
```

### Get Build Failure Details

Once you have a build ID:

```bash
# Get build details
az pipelines runs show --id <BUILD_ID> --org https://dev.azure.com/fluidframework --project public

# Get build timeline (shows individual task results)
curl -s "https://dev.azure.com/fluidframework/public/_apis/build/builds/<BUILD_ID>/timeline?api-version=7.1" | jq '.records[] | select(.result == "failed") | {name, type, result, log: .log.url}'
```

### Retrieve Build Logs

```bash
# List all logs for a build
az pipelines runs artifact list --run-id <BUILD_ID> --org https://dev.azure.com/fluidframework --project public

# Get specific log by ID from the timeline
curl -s "https://dev.azure.com/fluidframework/public/_apis/build/builds/<BUILD_ID>/logs/<LOG_ID>?api-version=7.1"
```

### Get Failed Task Logs

Combine timeline + log retrieval to get logs for failed steps:

```bash
# 1. Get the timeline to find failed tasks
TIMELINE=$(curl -s "https://dev.azure.com/fluidframework/public/_apis/build/builds/<BUILD_ID>/timeline?api-version=7.1")

# 2. Extract failed task log URLs
echo "$TIMELINE" | jq -r '.records[] | select(.result == "failed" and .log != null) | "\(.name): \(.log.url)"'

# 3. Fetch each log URL to see failure details
curl -s "<LOG_URL>"
```

## Tips

- Pipeline run IDs are different from build numbers. Use the run ID for API calls.
- The GitHub repository ID can be found via: `az repos list --org https://dev.azure.com/fluidframework --project public`
- For large log output, pipe through `tail` to see the most recent/relevant lines.
- Build results are: `succeeded`, `failed`, `canceled`, `partiallySucceeded`.
- Build statuses are: `notStarted`, `inProgress`, `completed`, `cancelling`, `postponed`.
