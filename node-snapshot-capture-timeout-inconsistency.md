# Node.js Tracer: Capture Timeout Documentation Bug

**Issue Type:** Documentation inconsistency / Potential bug
**Component:** Dynamic Instrumentation
**Environment Variable:** `DD_DYNAMIC_INSTRUMENTATION_CAPTURE_TIMEOUT_MS`
**Date Discovered:** 2026-02-20

## Summary

The Node.js tracer has an inconsistency between the documented default value and the actual default value for the Dynamic Instrumentation capture timeout setting.

- **Documented (TypeScript):** 100 milliseconds
- **Actual (Runtime):** 15 milliseconds

## Evidence

### TypeScript Definition (index.d.ts)

**Location:** `dd-trace-js/index.d.ts:1030`

```typescript
/**
 * Sets the timeout for capturing snapshot data in milliseconds. This timeout
 * ensures that expensive capture operations don't slow down the instrumented
 * application too much. If the timeout is exceeded, the capture operation is
 * aborted and an incomplete snapshot is sent.
 *
 * @default 100
 */
captureTimeoutMs?: number
```

The TypeScript documentation explicitly states `@default 100`.

### Actual Default (defaults.js)

**Location:** `dd-trace-js/packages/dd-trace/src/config/defaults.js:69`

```javascript
'dynamicInstrumentation.captureTimeoutMs': 15,
```

The actual runtime default is **15 milliseconds**, not 100.

### Configuration Mapping (index.js)

**Location:** `dd-trace-js/packages/dd-trace/src/config/index.js:289, 582-583`

```javascript
// Line 289 - Environment variable declaration
DD_DYNAMIC_INSTRUMENTATION_CAPTURE_TIMEOUT_MS,

// Lines 582-583 - Configuration mapping
target['dynamicInstrumentation.captureTimeoutMs'] = maybeInt(DD_DYNAMIC_INSTRUMENTATION_CAPTURE_TIMEOUT_MS)
unprocessedTarget['dynamicInstrumentation.captureTimeoutMs'] = DD_DYNAMIC_INSTRUMENTATION_CAPTURE_TIMEOUT_MS
```

The configuration correctly reads from the environment variable but falls back to the defaults.js value of 15ms when not set.

---

**Document Version:** 1.0
**Last Updated:** 2026-02-20
