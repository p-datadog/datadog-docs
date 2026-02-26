# dd-trace-py Logging Configuration Guide

## Overview

This guide explains how to configure logging for Datadog's Python tracer (dd-trace-py) to get only tracer logs or only Dynamic Instrumentation (DI) logs.

## Key Difference from Ruby Tracer

**Unlike dd-trace-rb**, Python's dd-trace-py:
- **Does NOT log full span details** by default (no pretty_inspect equivalent)
- Uses **component-specific log level control** via environment variables
- Implements **rate limiting** to prevent log spam (default: 1 log per 60 seconds)
- Uses **hierarchical logger prefix matching** for fine-grained control

---

## How DD_TRACE_DEBUG Works

### When `DD_TRACE_DEBUG=1` is enabled:

1. Sets all `ddtrace.*` loggers to **DEBUG level**
2. Enables file output when `DD_TRACE_LOG_FILE` is specified
3. Takes precedence over `DD_TRACE_LOG_LEVEL`
4. **Disables rate limiting** (all debug messages pass through)
5. Adds caller location to logs

**Important:** Unlike Ruby, this does NOT log full span details for every trace. Python logs are:
- Operational messages (span finish, context creation)
- Error conditions
- Component-specific debug info

---

## Environment Variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `DD_TRACE_DEBUG` | Enable debug logging for all components | `false` |
| `DD_TRACE_LOG_LEVEL` | Set global log level (NOTSET, DEBUG, INFO, WARNING, ERROR, CRITICAL) | `NOTSET` |
| `DD_TRACE_LOG_FILE` | Path to log file | None (logs to stderr) |
| `DD_TRACE_LOG_FILE_LEVEL` | Log level for file output | `DEBUG` |
| `DD_TRACE_LOG_STREAM_HANDLER` | Add stream handler | `true` |
| `DD_TRACE_LOGGING_RATE` | Rate limit seconds per unique log | `60` |
| `_DD_<COMPONENT>_LOG_LEVEL` | Component-specific log level | None |

---

## Component-Specific Log Level Control

dd-trace-py uses a **hierarchical prefix-based system** for controlling log levels per component.

### Pattern

```bash
_DD_<COMPONENT>_LOG_LEVEL=<LEVEL>
```

Where:
- `<COMPONENT>` is the module path with underscores (converts to dots)
- `<LEVEL>` is a Python log level (DEBUG, INFO, WARNING, ERROR, CRITICAL)

### How It Works

The logger name hierarchy is built from the environment variable:
1. Remove `_DD_` prefix and `_LOG_LEVEL` suffix
2. Convert underscores to dots
3. Match against `ddtrace.*` logger hierarchy

**Example:**
```bash
_DD_DEBUGGING_LOG_LEVEL=DEBUG
```
Applies to:
- `ddtrace.debugging`
- `ddtrace.debugging.submodule`
- `ddtrace.debugging.submodule.foo`

---

## Scenario 1: Only Tracer Logs (No DI Logs)

### Method A: Disable DI Logger

```bash
# Enable debug for tracer components
DD_TRACE_DEBUG=1

# Disable debug for DI component
_DD_DEBUGGING_LOG_LEVEL=WARNING

# Run application
python app.py
```

### Method B: Enable Specific Tracer Components Only

```bash
# Enable only span/tracer logging
_DD_TRACE_SPAN_LOG_LEVEL=DEBUG
_DD_TRACE_TRACER_LOG_LEVEL=DEBUG
_DD_TRACE_PROCESSOR_LOG_LEVEL=DEBUG

# Disable DI
_DD_DEBUGGING_LOG_LEVEL=WARNING

python app.py
```

### What You Get

```
DEBUG [ddtrace._trace.tracer] [tracer.py:588] finishing span - <Span(name='flask.request', ...)> (enabled:True)
DEBUG [ddtrace._trace.processor] [__init__.py:353] Starting span: <Span>, trace has 1 spans in the span aggregator
```

### What You DON'T Get

- DI probe execution messages
- DI signal collection messages
- DI uploader messages

---

## Scenario 2: Only DI Logs (No Tracer Logs)

### Method A: Enable DI Only

```bash
# Enable debug for DI component only
_DD_DEBUGGING_LOG_LEVEL=DEBUG

# Keep global level at INFO (or higher)
DD_TRACE_LOG_LEVEL=INFO

python app.py
```

### Method B: Disable Tracer Components

```bash
# Enable global debug
DD_TRACE_DEBUG=1

# Disable specific tracer components
_DD_TRACE_SPAN_LOG_LEVEL=WARNING
_DD_TRACE_TRACER_LOG_LEVEL=WARNING
_DD_TRACE_PROCESSOR_LOG_LEVEL=WARNING
_DD_TRACE_CONTEXT_LOG_LEVEL=WARNING

# Enable DI
_DD_DEBUGGING_LOG_LEVEL=DEBUG

python app.py
```

### What You Get

```
DEBUG [ddtrace.debugging._debugger] [_debugger.py:204] Enabling Debugger
DEBUG [ddtrace.debugging._debugger] [_debugger.py:258] Debugger enabled
DEBUG [ddtrace.debugging._signal.tracing] [tracing.py:82] Decorating span <Span> according to span decoration probe <Probe>
DEBUG [ddtrace.debugging._probe.registry] [registry.py:127] Registered probe: probe_id=abc123
```

### What You DON'T Get

- Span finishing messages
- Trace aggregation messages
- Span creation debug info

---

## Scenario 3: All Debug Logs

```bash
DD_TRACE_DEBUG=1
python app.py
```

**Output:** All ddtrace components log at DEBUG level

---

## Scenario 4: No Debug Logs

```bash
# Default behavior - no environment variables set
python app.py
```

Or explicitly:

```bash
DD_TRACE_DEBUG=0
DD_TRACE_LOG_LEVEL=WARNING
python app.py
```

---

## Logging Characteristics by Component

### Tracer (`ddtrace._trace.*`)

**What's logged at DEBUG:**
- `finishing span - <Span> (enabled:True)`
- `Starting span: <Span>, trace has N spans`
- `span closing after its parent, this is an error`
- Context creation/switching
- Sampling decisions

**What's NOT logged:**
- Full span details (tags, metrics, etc.)
- Span serialization
- No equivalent to Ruby's `pretty_inspect`

### Dynamic Instrumentation (`ddtrace.debugging.*`)

**What's logged at DEBUG:**
- Debugger enable/disable lifecycle
- Probe registration/unregistration
- Probe execution start/finish
- Signal collection
- Span decoration for probes
- Uploader queue operations

**What's NOT logged:**
- Captured snapshot data (sent to agent)
- Variable values (sent to agent)
- Stack traces (sent to agent)

### Profiling (`ddtrace.profiling.*`)

**What's logged at DEBUG:**
- Scheduler flush events
- Collector operations
- Stack sampling

### AppSec (`ddtrace.appsec.*`)

**What's logged at DEBUG:**
- WAF initialization
- Request context operations
- Rule matching
- IAST operations

---

## Advanced: Product-Based Structured Logging

dd-trace-py supports structured logging with a "product" field:

```python
from ddtrace.internal.logger import get_logger

log = get_logger(__name__)

# Structured log with product categorization
log.debug("operation::tag", extra={
    "product": "my_component",
    "more_info": ": additional context",
    "stack_limit": 4,  # Include 4 stack frames
    "exec_limit": 1    # Include 1 exception frame
}, stack_info=True)
```

**Output format:**
```
DEBUG my_component::operation::tag: additional context [2 skipped]
<stack trace here>
```

This is used internally by components like AppSec for categorized logging.

---

## Rate Limiting

**Default:** 1 log per 60 seconds per unique location (file:line)

**Disable rate limiting:**
```bash
DD_TRACE_LOGGING_RATE=0
```

**Increase rate limit:**
```bash
DD_TRACE_LOGGING_RATE=300  # 5 minutes
```

**Note:** Rate limiting is **automatically disabled** when logger is at DEBUG level.

---

## Summary Table

| Goal | Configuration | Notes |
|------|---------------|-------|
| **Only tracer logs** | `DD_TRACE_DEBUG=1` + `_DD_DEBUGGING_LOG_LEVEL=WARNING` | Disables DI debug output |
| **Only DI logs** | `_DD_DEBUGGING_LOG_LEVEL=DEBUG` + `DD_TRACE_LOG_LEVEL=INFO` | Enables DI only |
| **All debug logs** | `DD_TRACE_DEBUG=1` | All components at DEBUG |
| **No debug logs** | No env vars or `DD_TRACE_LOG_LEVEL=WARNING` | Default behavior |
| **Specific component** | `_DD_<COMPONENT>_LOG_LEVEL=DEBUG` | Fine-grained control |

---

## Key Insights

1. **No span pretty-print:** Unlike Ruby, Python does NOT log full span details by default
2. **Hierarchical control:** Use `_DD_*_LOG_LEVEL` for component-specific logging
3. **Rate limiting:** Prevents log spam (disabled at DEBUG level)
4. **Operational logs:** Python logs operational events, not data payloads
5. **DI uses signals:** DI doesn't log snapshot data; it uses a signal/uploader system

---

## Comparison with Ruby (dd-trace-rb)

| Feature | Ruby | Python |
|---------|------|--------|
| **DD_TRACE_DEBUG behavior** | Sets logger to DEBUG + enables span pretty_inspect | Sets logger to DEBUG only |
| **Span detail logging** | Yes (full pretty_inspect output) | No (operational messages only) |
| **Component-specific control** | Programmatic only | Via `_DD_*_LOG_LEVEL` env vars |
| **DI verbose logging** | `trace_logging` setting | `_DD_DEBUGGING_LOG_LEVEL=DEBUG` |
| **Rate limiting** | No | Yes (default 60s) |
| **Hierarchical logger control** | No | Yes (prefix trie matching) |

---

## Example Configurations

### Development: Verbose Everything
```bash
export DD_TRACE_DEBUG=1
export DD_TRACE_LOGGING_RATE=0  # Disable rate limiting
python app.py
```

### Production: Only Errors
```bash
export DD_TRACE_LOG_LEVEL=ERROR
python app.py
```

### Debug DI Issues Only
```bash
export _DD_DEBUGGING_LOG_LEVEL=DEBUG
export DD_TRACE_LOG_LEVEL=WARNING
python app.py
```

### Debug Specific Module
```bash
export _DD_TRACE_PROCESSOR_LOG_LEVEL=DEBUG
python app.py
```

---

## Code References

- Main logging config: `ddtrace/_logger.py`
- Internal logger: `ddtrace/internal/logger.py`
- Tracer logging: `ddtrace/_trace/tracer.py`
- Span logging: `ddtrace/_trace/span.py`
- DI logging: `ddtrace/debugging/_debugger.py`, `ddtrace/debugging/_signal/tracing.py`
- Documentation: `docs/contributing-testing.rst`
