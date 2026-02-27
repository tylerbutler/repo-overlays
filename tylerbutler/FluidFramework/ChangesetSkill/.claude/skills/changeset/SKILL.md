---
name: changeset
description: Create changesets for the Fluid Framework monorepo. Use when code changes need a changeset for release notes and changelogs. Triggers on mentions of changeset, changelog, release notes, or version bump.
---

# Changeset Creation

Create changesets for the Fluid Framework monorepo following project conventions.

## When a Changeset Is Needed

- Any PR that modifies published package behavior (features, bug fixes, deprecations, API changes)
- PRs labeled `changeset-required` are validated in CI
- Not needed for: test-only changes, internal tooling, documentation-only changes, or changes to unpublished packages

## File Format

Changesets live in `.changeset/` as markdown files with YAML front matter. File names use kebab-case words (e.g., `green-sides-watch.md`).

```markdown
---
"@fluidframework/tree": minor
"fluid-framework": minor
"__section": tree
---

Summary sentence describing the change

Optional longer description with details, migration guides, and code examples.
```

### Front Matter

**Package version bumps** (required): Map package names to bump types.

| Bump | When to Use |
|------|------------|
| `patch` | Bug fixes, internal refactors with no API change |
| `minor` | New features, new APIs, deprecations, promotions between release tags |
| `major` | Breaking changes (reserved for major releases; use `legacy` section for breaking legacy changes in minor releases) |

**Custom metadata** (prefixed with `__`):

| Key | Values | Default | Purpose |
|-----|--------|---------|---------|
| `__section` | `breaking`, `feature`, `tree`, `fix`, `deprecation`, `legacy`, `other` | (none) | Release notes section |
| `__highlight` | `true`, `false` | `false` | Highlight at top of section |
| `__includeInReleaseNotes` | `true`, `false` | `true` | Include in release notes |

### Release Notes Sections

| Section | Heading | Use For |
|---------|---------|---------|
| `breaking` | Breaking Changes | Major release breaking changes only |
| `feature` | New Features | New APIs, capabilities |
| `tree` | SharedTree DDS Changes | Anything in `@fluidframework/tree` or SharedTree-related |
| `fix` | Bug Fixes | Bug fixes |
| `deprecation` | Deprecations | Deprecating APIs |
| `legacy` | Legacy API Changes | Breaking changes to legacy/deprecated APIs in minor releases |
| `other` | Other Changes | Everything else |

### Affected Packages

- All packages in the client release group are versioned together (fixed group)
- Always include **both** the specific package and `fluid-framework` if the change affects APIs re-exported through `fluid-framework`
- Scopes excluded by default: `@fluid-example`, `@fluid-internal`, `@fluid-test`

### Markdown Body

**First paragraph** = summary (single sentence, used as changelog entry title).

**Subsequent paragraphs** = body (optional details, migration guides, code examples).

## Writing Conventions

### Style Rules (enforced by Vale)

- Use descriptive, customer-facing language
- Use proper capitalization for framework terms:
  - **Correct**: SharedTree, Fluid Framework, FluidFramework, DDSes, TypeScript, JavaScript
  - **Incorrect**: shared tree, fluid, fluid framework
- Write in active voice when possible
- Include migration paths when deprecating or changing APIs
- Code examples should use TypeScript with proper syntax highlighting

### Content Guidelines

- **Feature additions**: Describe the new API/capability, show usage example
- **Bug fixes**: Describe what was broken and what the fix does
- **Deprecations**: List deprecated items, explain replacements, show migration code
- **Promotions** (alpha to beta, beta to public): State what was promoted and to which release tag
- **Breaking changes**: Describe what changed and how to migrate

### Code Examples

Use fenced TypeScript blocks with proper imports:

````markdown
```typescript
import { TreeAlpha } from "@fluidframework/tree/alpha";

const context = TreeAlpha.context(node);
```
````

Link to API docs when referencing specific APIs:
```markdown
[`checkSchemaCompatibilitySnapshots`](https://fluidframework.com/docs/api/fluid-framework#checkschemacompatibilitysnapshots-function)
```

## Creating the File

### Option 1: Write directly

Create a new file in `.changeset/` with a descriptive kebab-case name:

```bash
# Name should describe the change
# e.g., add-context-api.md, deprecate-old-collection.md, fix-tree-rebase.md
```

### Option 2: Interactive CLI

```bash
pnpm changeset
# Equivalent to: flub changeset add --releaseGroup client
```

The CLI will prompt for package selection, section, summary, and description.

## Validation

Changesets are validated in CI:

1. **Presence check**: `flub check changeset` verifies a changeset exists for PRs with the `changeset-required` label
2. **Vale linting**: `pnpm run check:changesets` runs Vale style checks against changeset files
3. **Reporter**: A GitHub bot comments on PRs about changeset status

Run locally before pushing:

```bash
pnpm run check:changesets
```

## Examples

### New Feature

```markdown
---
"@fluidframework/tree": minor
"fluid-framework": minor
"__section": feature
---

Added new TreeAlpha.context(node) API

This release introduces a node-scoped context that works for both hydrated and unhydrated TreeNodes.

#### Migration

If you previously used `TreeAlpha.branch(node)`, switch to `TreeAlpha.context(node)`:

\```typescript
import { TreeAlpha } from "@fluidframework/tree/alpha";

const context = TreeAlpha.context(node);
if (context.isBranch()) {
    context.fork();
}
\```
```

### Deprecation

```markdown
---
"@fluidframework/register-collection": minor
"@fluidframework/ordered-collection": minor
"__section": deprecation
---

Deprecated DDS implementation classes

The following DDS implementation classes are now deprecated and will be removed in a future release:

- `ConsensusRegisterCollectionClass` — use `ConsensusRegisterCollectionFactory` to create instances
- `ConsensusOrderedCollection` — use `IConsensusOrderedCollection` for typing
- `ConsensusQueueClass` — use the `ConsensusQueue` singleton to create instances
```

### Bug Fix

```markdown
---
"@fluidframework/tree": patch
"fluid-framework": patch
"__section": fix
---

Fixed rebase error when rebasing over a schema change

Previously, rebasing a transaction over a schema change could throw an unexpected error. This has been fixed.
```

### Promotion to Beta

```markdown
---
"fluid-framework": minor
"@fluidframework/tree": minor
"__section": tree
---

Promote checkSchemaCompatibilitySnapshots to beta

[`checkSchemaCompatibilitySnapshots`](https://fluidframework.com/docs/api/fluid-framework#checkschemacompatibilitysnapshots-function) has been promoted to `@beta`.
It is recommended that all SharedTree applications use this API to write schema compatibility tests.
```
