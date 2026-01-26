---
name: changeset
description: Write changesets for the Fluid Framework repository
---

# Writing Changesets

Create changeset files to document changes for the changelog and release notes.

## File Location and Naming

Changesets are markdown files stored in `.changeset/`. Use descriptive kebab-case filenames (e.g., `fix-container-reconnection.md`).

## File Format

Changesets have YAML front matter followed by markdown content:

```markdown
---
"@fluidframework/package-name": minor
"__section": feature
---

Summary heading in present tense

Detailed description of the change. Explain what changed and why it benefits users.
Include code examples if helpful.
```

## Front Matter Fields

### Package Metadata (Required)

List affected packages with their bump type:
- `"@fluidframework/package-name": patch` - Bug fixes, documentation
- `"@fluidframework/package-name": minor` - New features, non-breaking changes
- `"@fluidframework/package-name": major` - Breaking changes (rare, mainly server)

### Custom Metadata (Optional, prefixed with `__`)

| Field | Values | Description |
|-------|--------|-------------|
| `__section` | `breaking`, `feature`, `tree`, `fix`, `deprecation`, `legacy`, `other` | Release notes section |
| `__includeInReleaseNotes` | `true`/`false` | Include in release notes (default: true) |
| `__highlight` | `true`/`false` | Feature as highlighted change (default: false) |

### Section Guidelines

- **breaking**: Major breaking changes (use for major releases, primarily server)
- **feature**: New features and capabilities
- **tree**: SharedTree DDS-specific changes
- **fix**: Bug fixes
- **deprecation**: API deprecations
- **legacy**: Breaking changes to legacy APIs (for client releases)
- **other**: Changes that don't fit other categories

## Content Requirements

### Structure

1. **Summary line** (required): First paragraph, serves as the heading
2. **Body** (required): Detailed description (minimum 2 paragraphs total)

### Writing Style

- Use present or present-perfect tense ("is", "has been", not "was" or "will be")
- Summary should be succinct and benefit-focused
- No code formatting (backticks) in the summary heading
- No terminal punctuation on the summary line
- Start markdown headings at level 4 (`####`) if needed in body
- Include documentation links where relevant
- Never submit just a link; always include explanatory text

### Bad vs Good Examples

**Bad:**
```markdown
---
"@fluidframework/container-runtime": minor
---

Fixed the bug
```

**Good:**
```markdown
---
"@fluidframework/container-runtime": minor
"__section": fix
---

Container reconnection is more reliable

The container now properly handles reconnection when the network temporarily drops. Previously, rapid network changes could cause the container to enter an inconsistent state.

Applications that experienced intermittent disconnection issues should see improved stability.
```

## Creating Changesets

### Manual Creation (Preferred)

Create a new file directly in `.changeset/` with a descriptive name:

1. Choose a descriptive kebab-case filename (e.g., `fix-reconnection-handling.md`)
2. Add the YAML front matter with affected packages and bump types
3. Add the `__section` metadata for release notes categorization
4. Write the summary line and detailed body
5. Save the file in `.changeset/`

### Finding Affected Packages

To determine which packages are affected by your changes:

```bash
# See changed files
git diff --name-only main

# Or check git status for uncommitted changes
git status
```

Map changed files to their package names by checking the `package.json` in each package directory.

### Interactive Tool (User Only)

Users can run the interactive changeset tool themselves:

```bash
pnpm flub changeset add
```

This command is interactive and requires user input, so Claude cannot run it directly.

## Workflow

1. Make your code changes
2. Create a changeset file manually in `.changeset/`
3. List all affected packages with appropriate bump types
4. Write a clear summary and detailed description
5. Set the appropriate `__section` for release notes categorization
6. Commit the changeset file with your code changes

## Tips

- One changeset can cover multiple packages if they're changed together for the same reason
- The first package listed is considered the "main" package
- Changesets are consumed during release and automatically deleted
- You can edit changesets after creation - both content and bump types
- If a pre-commit hook or CI fails, fix issues and create a new commit (don't amend)
- Package names must be quoted in the YAML front matter
- Custom metadata keys must be quoted and prefixed with `__`
