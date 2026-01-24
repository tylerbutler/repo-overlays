# CLAUDE.md - @fluidframework/tree-agent-langchain

## Purpose

This package provides LangChain integration for `@fluidframework/tree-agent`. It implements the `SharedTreeChatModel` interface using LangChain's `BaseChatModel`, allowing you to use any LangChain-compatible LLM (OpenAI, Anthropic, Google Gemini, etc.) with the `SharedTreeSemanticAgent` to perform AI-driven edits on SharedTree data structures.

## Architecture

### Core Component

**`createLangchainChatModel(langchainModel: BaseChatModel): SharedTreeChatModel`** (alpha export)

Creates an adapter that wraps a LangChain chat model to work with `SharedTreeSemanticAgent`. The adapter:

1. Manages conversation history as LangChain `BaseMessage` objects (SystemMessage, HumanMessage)
2. Registers a tool named `GenerateTreeEditingCode` that the LLM uses to submit JavaScript code for tree edits
3. Handles the edit loop - when the LLM calls the tool, executes the edit and recurses until the LLM stops calling tools or hits the edit limit

### Message Flow

```
User Query
    |
    v
SharedTreeSemanticAgent (tree-agent)
    |
    | appendContext() - schema info, domain hints
    | query() - user prompt
    v
LangchainChatModel (this package)
    |
    | Binds GenerateTreeEditingCode tool
    | Sends messages to LLM
    v
LangChain BaseChatModel
    |
    | LLM generates tool call with JS code
    v
LangchainChatModel
    |
    | Invokes edit() callback with generated code
    | Recurses until LLM stops or max edits
    v
Response text
```

### Key Implementation Details

- **Tool binding**: Uses LangChain's `tool()` helper with Zod schema for the `functionCode` parameter
- **Conversation persistence**: The `messages` array accumulates across multiple queries, maintaining context
- **Edit loop**: After each tool call, the result is appended to history and the model is queried again
- **Exit conditions**: Loop terminates when LLM returns text without tool calls, or when `tooManyEditsError` is returned

## Testing

Tests are integration tests against live LLM APIs and are **skipped by default** (`describe.skip`).

### Running Integration Tests

Requires API keys as environment variables:
- `OPENAI_API_KEY` - for OpenAI/GPT models
- `ANTHROPIC_API_KEY` - for Claude models
- `GEMINI_API_KEY` - for Google Gemini models

```bash
# Remove .skip from describe() in integration.spec.ts, then:
pnpm test:mocha
```

### Test Structure

- **Domains** (`src/test/domains/`): SharedTree schemas for test scenarios (Users, Groceries, Conference, etc.)
- **Scenarios** (`src/test/scenarios/`): Individual test cases with initial data, prompts, and expected results
- **Scoring**: Tests use a custom scoring system (`scoreSymbol`) to evaluate LLM output quality rather than exact matching

### Test Scenarios

| Scenario | Description |
|----------|-------------|
| `addUsers` | Add new entries to a map-based user collection |
| `updateUser` | Modify existing user properties |
| `sortGroceries` | Reorder array items |
| `addComment` | Add text content |
| `methodUse` | Test LLM calling exposed schema methods |

Results are written to `src/test/integration-test-results/` with per-provider markdown logs.

## Exports

| Export Path | Content |
|-------------|---------|
| `.` (public) | No public exports currently |
| `./alpha` | `createLangchainChatModel` |
| `./internal` | All exports |
