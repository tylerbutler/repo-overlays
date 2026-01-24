# @fluidframework/app-insights-logger

Telemetry logger that routes Fluid Framework telemetry events to Azure App Insights using the `trackEvent` API.

## Purpose

Provides an `ITelemetryBaseLogger` implementation that forwards Fluid telemetry to Azure App Insights with configurable filtering. This allows applications to selectively send telemetry based on event category and/or namespace patterns.

## Key Concepts

### Filter Modes

- **Exclusive mode**: All events are sent by default; matching filters exclude events
- **Inclusive mode**: No events are sent by default; matching filters include events

### Filter Types

Filters can match on:
- **Category**: `generic`, `error`, `performance`
- **Namespace**: Prefix matching on `eventName` (e.g., `"perf:latency"` matches `"perf:latency:ops"`)
- **Combined**: Both category AND namespace must match

### Filter Specificity

More specific filters take precedence. Filters are sorted by namespace length (longest first), so `"A:B:C"` is evaluated before `"A:B"`.

### Namespace Exceptions

Filters can define `namespacePatternExceptions` to exclude child namespaces from the filter match.

## Main Exports

```typescript
import {
  createLogger,
  type FluidAppInsightsLoggerConfig,
  type TelemetryFilter,
  type CategoryFilter,
  type NamespaceFilter,
  type TelemetryEventCategory,
} from "@fluidframework/app-insights-logger/beta";
```

## Usage

```typescript
import { ApplicationInsights } from "@microsoft/applicationinsights-web";
import { createLogger } from "@fluidframework/app-insights-logger/beta";

const appInsights = new ApplicationInsights({ config: { connectionString: "..." } });
appInsights.loadAppInsights(); // Required before logging works

// Example: Only send error and performance events from specific namespaces
const logger = createLogger(appInsights, {
  filtering: {
    mode: "inclusive",
    filters: [
      { categories: ["error"] }, // All errors
      {
        namespacePattern: "perf:latency",
        categories: ["performance"],
        namespacePatternExceptions: new Set(["perf:latency:ops"]),
      },
    ],
  },
});
```

## Filter Validation

The constructor validates filters and throws on:
- Duplicate `namespacePattern` values
- Multiple category-only filters (would be ambiguous)
- Namespace exceptions that aren't children of their parent pattern

## Testing

Tests use Sinon to spy on `ApplicationInsights.trackEvent()` calls. The test file covers:
- Basic event routing
- Filter validation errors
- Both inclusive and exclusive modes
- Category, namespace, and combined filtering
- Namespace exception handling
- Filter specificity ordering

Run tests with `pnpm test:mocha` from the package directory.

## Implementation Notes

- Config is deep-cloned on construction to prevent mutation issues
- All exports are `@beta` release tag
- Depends on `@ungap/structured-clone` for config cloning
