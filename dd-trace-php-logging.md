# dd-trace-php Logging Configuration Guide

## Overview

This guide explains how to configure logging for Datadog's PHP tracer (dd-trace-php) to control tracer, Dynamic Instrumentation (DI), and component logging output.

## Key Characteristics

**Unlike other Datadog tracers**, dd-trace-php:
- **Logs startup configuration as JSON** once per process
- **Does NOT log full span details** by default (only trace_id/span_id)
- Implements **automatic log file rotation** (60 seconds)
- Uses **component-based LOG() macro** system for categorization
- Implements **trace-level rate limiting** (100 traces/sec default)
- Written in **C extension + Rust FFI** for performance
- Supports **runtime INI changes** (non-CLI SAPIs only)

---

## How DD_TRACE_DEBUG Works

### When `DD_TRACE_DEBUG=1` is enabled:

**Location:** `/ext/configuration.h:127`, `/ext/logging.c:228-234`

```c
CONFIG(BOOL, DD_TRACE_DEBUG, "false", .ini_change = ddtrace_alter_dd_trace_debug)

bool ddtrace_alter_dd_trace_debug(zval *old_value, zval *new_value, zend_string *new_str) {
    dd_log_set_level(Z_TYPE_P(new_value) == IS_TRUE);
    return true;
}
```

**Behavior:**
1. Enables **DEBUG level and higher** (TRACE, DEBUG, INFO, WARN, ERROR)
2. Logs **startup configuration JSON** (once per process)
3. Logs **span lifecycle** with trace_id/span_id
4. **Does NOT log full span details** (tags, metrics, etc.)
5. SAPI-specific adjustments:
   - Non-CLI with startup logs: Full `"debug"` output
   - CLI or without startup logs: `"debug,startup=error"` (demotes startup to error level)
6. Enables **automatic log file rotation** (60s)
7. Can be changed at **runtime via INI** (non-CLI SAPIs only)

**Important:** Unlike Ruby, this does NOT log complete span data. PHP logs are lightweight and focus on:
- Startup configuration (JSON, once per process)
- Span lifecycle events (trace_id, span_id, parent_id)
- Error conditions
- Excluded module warnings
- Telemetry status

---

## Environment Variables

| Variable | Purpose | Default | Location |
|----------|---------|---------|----------|
| `DD_TRACE_DEBUG` | Enable debug logging for all components | `false` | `/ext/configuration.h:127` |
| `DD_TRACE_LOG_LEVEL` | Set global log level (error, warn, info, debug, trace) | `error` | `/ext/configuration.h:234` |
| `DD_TRACE_LOG_FILE` | Path to log file | None (stderr) | `/ext/configuration.h:236` |
| `DD_TRACE_STARTUP_LOGS` | Enable startup configuration JSON dump | `true` | `/ext/configuration.h:203` |
| `DD_TRACE_ONCE_LOGS` | Log once-only messages (LOG_ONCE flag) | `true` | `/ext/configuration.h:207` |
| `DD_TRACE_RATE_LIMIT` | Max traces per second | `100` | `/ext/configuration.h:163` |
| `DD_TRACE_AGENT_DEBUG_VERBOSE_CURL` | Verbose CURL output for agent communication | `false` | Config |
| `DD_TRACE_DEBUG_CURL_OUTPUT` | Enable curl debug output (background sender) | `false` | Config |
| `DD_DYNAMIC_INSTRUMENTATION_ENABLED` | Enable Dynamic Instrumentation | `false` | `/ext/configuration.h:253` |
| `DD_DYNAMIC_INSTRUMENTATION_REDACTED_IDENTIFIERS` | Redact DI variable names (set) | Empty | `/ext/configuration.h:254` |
| `DD_DYNAMIC_INSTRUMENTATION_REDACTED_TYPES` | Redact DI types (set) | Empty | `/ext/configuration.h:256` |

---

## Log Levels and Component System

### Hierarchical Log Levels

**Defined in:** `/components-rs/common.h`

```c
typedef enum ddog_Log {
    DDOG_LOG_ERROR = 1,
    DDOG_LOG_WARN = 2,
    DDOG_LOG_INFO = 3,
    DDOG_LOG_DEBUG = 4,
    DDOG_LOG_TRACE = 5,
    
    // Special levels with flags
    DDOG_LOG_DEPRECATED = (3 | ddog_LOG_ONCE),          // INFO + once-only
    DDOG_LOG_STARTUP = (3 | (2 << 4)),                  // INFO + startup component
    DDOG_LOG_STARTUP_WARN = (1 | (2 << 4)),             // ERROR + startup component
    DDOG_LOG_SPAN = (4 | (3 << 4)),                     // DEBUG + span component
    DDOG_LOG_SPAN_TRACE = (5 | (3 << 4)),               // TRACE + span component
    DDOG_LOG_HOOK_TRACE = (5 | (4 << 4)),               // TRACE + hook component
} ddog_Log;
```

**Bitfield format:** `level | (component_id << 4) | flags`

### Component Categories

**Internal component IDs:**
- **0**: DEFAULT (all levels)
- **2**: STARTUP (INFO, WARN)
- **3**: SPAN (DEBUG, TRACE)
- **4**: HOOK (TRACE)

**LOG() Macro:**
```c
#define LOG(source, format, ...) \
    do { \
        if (ddog_shall_log(DDOG_LOG_##source)) { \
            ddog_logf(DDOG_LOG_##source, (DDOG_LOG_##source & ddog_LOG_ONCE) \!= 0, \
                     format, ##__VA_ARGS__); \
        } \
    } while (0)
```

**Usage examples:**
```c
LOG(SPAN_TRACE, "Starting span: trace_id=%s, span_id=%" PRIu64, ...);
LOG(ERROR, "Module conflict detected: %s", module_name);
LOG(STARTUP, "Configuration loaded successfully");
```

---

## Startup Configuration Logging

### JSON Configuration Dump

**Location:** `/ext/startup_logging.c:128-190`

**When logged:**
- First request only (controlled by `DD_TRACE_ONCE_LOGS`)
- Enabled by `DD_TRACE_STARTUP_LOGS=true` (default)
- Logged at INFO level with STARTUP component

**Format:**
```
DATADOG TRACER CONFIGURATION - {"date":"2024-02-25T10:30:45Z",...}
```

**JSON fields:**
- `date`: ISO 8601 timestamp
- `os_name`, `os_version`: Operating system info
- `version`: Tracer version
- `lang`: "php"
- `lang_version`: PHP version (e.g., "8.3.0")
- `env`: DD_ENV value
- `enabled`: Tracing enabled status
- `service`: Service name
- `agent_url`: Datadog agent endpoint
- `debug`: Debug flag status
- `analytics_enabled`: Analytics status
- `sample_rate`: Trace sampling rate
- `sampling_rules`: JSON sampling rules
- `tags`: Global tags object
- `service_mapping`: Service name mappings
- `distributed_tracing_enabled`: Distributed tracing status
- `priority_sampling_enabled`: Priority sampling status
- `dd_version`: DD_VERSION tag
- `architecture`: System architecture
- `sapi`: PHP SAPI (cli, fpm-fcgi, apache2handler, etc.)
- Additional runtime configuration...

**Size:** ~500-1000 characters (single line)

---

## Span Lifecycle Logging

### What Gets Logged

**Location:** `/ext/span.c:54-980`

**Logged at SPAN_TRACE level:**
```c
LOG(SPAN_TRACE, "Starting new root span: trace_id=%s, span_id=%" PRIu64 
    ", parent_id=%" PRIu64 ", SpanStack=%d, parent_SpanStack=%d", 
    Z_STRVAL(root->property_trace_id), span->span_id, root->parent_id, 
    root->stack->std.handle, root->stack->parent_stack->std.handle);

LOG(SPAN_TRACE, "Closing span: trace_id=%s, span_id=%" PRIu64, 
    Z_STRVAL(root->property_trace_id), span->span_id);

LOG(SPAN_TRACE, "Dropping span: trace_id=%s, span_id=%" PRIu64, 
    Z_STRVAL(root->property_trace_id), span->span_id);
```

**What is NOT logged:**
- Span tags (e.g., `http.method`, `http.url`)
- Span metrics (e.g., `_dd.top_level`, `_sampling_priority_v1`)
- Span resource names
- Span type
- Span duration
- Full span serialization

**Philosophy:** Minimal logging for production performance; use APM UI to inspect span data.

---

## Configuration Scenarios

### Scenario 1: Only Tracer Logs (No DI)

```bash
# Enable tracer debug, disable DI
export DD_TRACE_DEBUG=1
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=0
export DD_TRACE_LOG_FILE=/var/log/datadog-tracer.log

php app.php
```

**What you get:**
- Startup configuration JSON (once)
- Span lifecycle messages
- Error/warning messages
- No DI probe logs

---

### Scenario 2: Only DI Logs (Tracer at Higher Level)

```bash
# Enable DI, keep tracer at WARN level
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=1
export DD_TRACE_LOG_LEVEL=warn
export DD_TRACE_LOG_FILE=/var/log/datadog-di.log

php app.php
```

**What you get:**
- DI probe lifecycle messages
- DI snapshot collection messages
- Tracer errors/warnings only
- No span TRACE-level logs

**Note:** PHP doesn't have separate DI log level control; DI and tracer use the same logger.

---

### Scenario 3: Full Debug (Development)

```bash
# Everything at DEBUG level
export DD_TRACE_DEBUG=1
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=1
export DD_TRACE_LOG_FILE=/tmp/dd-debug.log
export DD_TRACE_ONCE_LOGS=0  # Allow repeated logs
export DD_TRACE_STARTUP_LOGS=1

php app.php
```

**What you get:**
- Startup configuration JSON
- All span lifecycle events
- DI probe execution logs
- Hook instrumentation traces
- All warnings and errors

---

### Scenario 4: Production (Minimal Noise)

```bash
# Errors only, with startup JSON
export DD_TRACE_LOG_LEVEL=error
export DD_TRACE_STARTUP_LOGS=1  # Keep startup JSON (once only)
export DD_TRACE_LOG_FILE=/var/log/datadog-php.log
export DD_TRACE_RATE_LIMIT=100

php-fpm
```

**What you get:**
- Startup configuration JSON (first request)
- Error messages only
- Rate-limited traces (100/sec max)
- No debug or info logs

---

### Scenario 5: Background Sender Debugging

```bash
# Debug agent communication
export DD_TRACE_DEBUG=1
export DD_TRACE_AGENT_DEBUG_VERBOSE_CURL=1
export DD_TRACE_DEBUG_CURL_OUTPUT=1
export DD_TRACE_LOG_FILE=/var/log/datadog-curl.log

php app.php
```

**What you get:**
- Full CURL request/response logs
- Agent communication details
- HTTP headers and payloads
- Network timing information

---

## Rate Limiting

### Trace-Level Rate Limiting

**Location:** `/ext/limiter/limiter.c`

**Configuration:**
```bash
export DD_TRACE_RATE_LIMIT=100  # Max 100 traces per second
```

**Implementation:**
```c
typedef struct {
    uint32_t limit;           // from DD_TRACE_RATE_LIMIT config
    struct {
        _Atomic(int64_t) hit_count;
        _Atomic(uint64_t) last_update;
        _Atomic(int64_t) recent_total;
    } window;
} ddtrace_limiter;
```

**Mechanism:**
- **Sliding window counter** with nanosecond precision (`zend_hrtime()`)
- **Atomic operations** for thread-safety across forked processes
- **Shared anonymous memory** for inter-process coordination
- Allows configured traces/sec (default: 100)
- Integrated with APM tracing control (`DD_APM_TRACING_ENABLED`)

**Algorithm:** `/ext/limiter/limiter.c:63-80`
- Compares current time window hit count against limit
- Updates window atomically on transitions
- Prevents race conditions in multi-process environments (FPM, Apache workers)

### Exception Hash Rate Limiting

**Purpose:** Prevent duplicate exception captures

**Location:** `/ext/exception_serialize.c:386`
```c
ddog_exception_hash_limiter_inc()
```

**Mechanism:**
- Hash-based deduplication
- Granularity in seconds
- Prevents redundant exception snapshots

### Log Once Flag

**Configuration:**
```bash
export DD_TRACE_ONCE_LOGS=1  # Default: true
```

**How it works:**
- Messages marked with `LOG_ONCE` flag only output once per process
- Bitwise check: `(DDOG_LOG_source & ddog_LOG_ONCE) \!= 0`
- Examples: `DDOG_LOG_DEPRECATED`, startup logs

**Disable for debugging:**
```bash
export DD_TRACE_ONCE_LOGS=0  # Allow repeated logs
```

---

## Automatic Log File Rotation

### Atomic Rotation Mechanism

**Location:** `/ext/logging.c:124-165`

**How it works:**
1. **Every 60 seconds**: Checks if rotation is needed
2. **Atomic file descriptor rotation**: Prevents race conditions
3. **Automatic reopening**: Opens new file descriptor
4. **Timestamp prefixing**: `[dd-Mon-YYYY HH:MM:SS TZ]`

**Supported destinations:**
- File paths: `/var/log/datadog-php.log`
- PHP streams: `php://stderr`, `php://stdout`
- Fallback: PHP `error_log` directive

**Configuration:**
```bash
export DD_TRACE_LOG_FILE=/var/log/datadog.log
```

**Benefits:**
- No manual log rotation required
- Works with logrotate (graceful reopening)
- Thread-safe across forked processes
- Automatic in production environments

---

## Dynamic Instrumentation (DI) Logging

### Configuration

**Location:** `/ext/configuration.h:253-256`

```bash
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=1
export DD_DYNAMIC_INSTRUMENTATION_REDACTED_IDENTIFIERS="password,secret"
export DD_DYNAMIC_INSTRUMENTATION_REDACTED_TYPES="SensitiveData"
```

### DI Logging Functions

**Location:** `/ext/ddtrace.h` (Rust FFI)

```c
void ddog_log_debugger_data(const struct ddog_Vec_DebuggerPayload *payloads);
void ddog_log_debugger_datum(const struct ddog_DebuggerPayload *payload);
ddog_MaybeError ddog_send_debugger_diagnostics(
    const struct ddog_RemoteConfigState *remote_config_state, ...);
```

### Live Debugger Components

**Location:** `/ext/live_debugger.h:7-12`, `/ext/live_debugger.c`

- `ddtrace_live_debugger_setup` - Configuration setup
- `ddtrace_live_debugger_minit()` - Module initialization
- `ddtrace_live_debugger_mshutdown()` - Module shutdown
- Probe evaluation and value capture with redaction support
- Snapshot-based variable capture

**What gets logged:**
- Probe registration/unregistration
- Probe execution start/finish
- Snapshot collection events
- Redaction operations
- Debugger lifecycle messages

**What does NOT get logged:**
- Captured variable values (sent to agent)
- Snapshot data payloads (sent to agent)
- Stack traces (sent to agent)

---

## JSON Logging Support

### Extension Logs (C Side)

**Format:** Plain text with timestamps

```
[dd-Mon-2024 10:30:45 UTC] [SPAN_TRACE] Starting span: trace_id=1234567890123456789
```

### Userland API (PHP Side)

**Full JSON support via DatadogLogger**

**Location:** `/src/api/Log/DatadogLogger.php:131-175`

```php
use DDTrace\Log\DatadogLogger;
use DDTrace\Log\PsrLogger;

// Structured JSON logging
$logger = new DatadogLogger('/var/log/app.log');
$logger->info('User action', ['user_id' => 123, 'action' => 'login']);
```

**Output format:**
```json
{
  "message": "User action",
  "status": "info",
  "timestamp": "2024-02-25T10:30:45.123456+00:00",
  "user_id": 123,
  "action": "login",
  "dd.trace_id": "1234567890123456789",
  "dd.span_id": "9876543210987654321"
}
```

**Log Injection (DD_LOGS_INJECTION):**

**Location:** `/src/api/Log/DatadogLogger.php:160-175`

```php
private static function handleLogInjection(): array {
    if (\dd_trace_env_config('DD_LOGS_INJECTION')) {
        $traceId = \DDTrace\logs_correlation_trace_id();
        $spanId = \dd_trace_peek_span_id();
        if ($traceId && $spanId) {
            return [
                'dd.trace_id' => $traceId,
                'dd.span_id' => $spanId,
            ];
        }
    }
    return [];
}
```

**Enable:**
```bash
export DD_LOGS_INJECTION=1
```

**Result:** All logs automatically include trace and span IDs for correlation.

---

## Runtime INI Changes

### Non-CLI SAPIs Only

**Supported SAPIs:**
- Apache (`apache2handler`, `mod_php`)
- FPM (`fpm-fcgi`)
- CGI (`cgi-fcgi`)
- Nginx with FPM

**Not supported:**
- CLI (`cli`) - requires restart

### Example Usage

```php
// Enable debug for single request (web servers only)
ini_set('datadog.trace.debug', '1');

// ... debug code here ...

// Disable debug
ini_set('datadog.trace.debug', '0');
```

**Mechanism:**

**Location:** `/ext/logging.c:228-234`, `/ext/configuration.h`

```c
bool ddtrace_alter_dd_trace_debug(zval *old_value, zval *new_value, zend_string *new_str) {
    dd_log_set_level(Z_TYPE_P(new_value) == IS_TRUE);
    return true;
}

bool ddtrace_alter_dd_trace_log_level(zval *old_value, zval *new_value, zend_string *new_str) {
    if (get_DD_TRACE_DEBUG()) {
        return true;  // DD_TRACE_DEBUG takes precedence
    }
    ddog_set_log_level(dd_zend_string_to_CharSlice(Z_STR_P(new_value)), ...);
    return true;
}
```

**INI change callbacks** trigger log level adjustments immediately.

---

## Verbosity Comparison

### Per Request (3 spans)

**First request with DD_TRACE_DEBUG=1:**
- Startup JSON: ~1 line (500-1000 chars)
- Root span start: 1 line
- Child span starts: ~1-2 lines
- Span closes: ~1-3 lines

**Total:** ~5-8 lines (+ one-time startup)

**Subsequent requests:**
- Span lifecycle only: ~3-8 lines per trace

### Compared to Other Tracers

| Tracer | Lines per trace | Includes span details? |
|--------|----------------|------------------------|
| Ruby | ~100-150 | ✅ Yes (full pretty_inspect) |
| Python | ~10-20 | ❌ No (operational only) |
| Java | ~5-10 | ⚠️ Optional (TraceStructureWriter) |
| **PHP** | **~3-8** | **❌ No (trace_id/span_id only)** |

**PHP is the most lightweight** - logs only essential lifecycle events with trace/span IDs.

---

## Best Practices

### Development

```bash
# Full debug with all messages
export DD_TRACE_DEBUG=1
export DD_TRACE_LOG_FILE=/tmp/dd-trace-debug.log
export DD_TRACE_STARTUP_LOGS=1
export DD_TRACE_ONCE_LOGS=0  # Allow repeated logs
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=1

php app.php
```

### Production

```bash
# Conservative - errors only
export DD_TRACE_LOG_LEVEL=error
export DD_TRACE_STARTUP_LOGS=1  # Keep config JSON (once only)
export DD_TRACE_LOG_FILE=/var/log/datadog-php.log
export DD_TRACE_RATE_LIMIT=100  # Limit to 100 traces/sec
export DD_LOGS_INJECTION=1      # Enable log correlation

# Run with your web server
php-fpm
```

### Temporary Debugging (Non-CLI)

```php
// Enable debug for a single request (web servers only)
ini_set('datadog.trace.debug', '1');

// ... debug code here ...

// Disable debug
ini_set('datadog.trace.debug', '0');
```

### Background Sender Debugging

```bash
# Verbose CURL output for agent communication
export DD_TRACE_DEBUG=1
export DD_TRACE_AGENT_DEBUG_VERBOSE_CURL=1
export DD_TRACE_DEBUG_CURL_OUTPUT=1
export DD_TRACE_LOG_FILE=/var/log/datadog-curl.log

php app.php
```

### Docker/Container Logging

```bash
# Log to stdout for container log collection
export DD_TRACE_DEBUG=1
export DD_TRACE_LOG_FILE=php://stdout

docker run -e DD_TRACE_DEBUG=1 -e DD_TRACE_LOG_FILE=php://stdout myapp
```

---

## Common Use Cases

### 1. Debug Startup Configuration

```bash
export DD_TRACE_STARTUP_LOGS=1
export DD_TRACE_LOG_FILE=/tmp/startup.log
php app.php
```

**Look for:** JSON configuration dump with all settings.

### 2. Investigate Span Sampling Issues

```bash
export DD_TRACE_DEBUG=1
export DD_TRACE_LOG_LEVEL=trace
export DD_TRACE_LOG_FILE=/tmp/spans.log
php app.php
```

**Look for:** `SPAN_TRACE` messages with trace_id/span_id.

### 3. Troubleshoot Agent Communication

```bash
export DD_TRACE_AGENT_DEBUG_VERBOSE_CURL=1
export DD_TRACE_DEBUG_CURL_OUTPUT=1
export DD_TRACE_LOG_FILE=/tmp/agent.log
php app.php
```

**Look for:** CURL request/response logs.

### 4. Monitor DI Probe Execution

```bash
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=1
export DD_TRACE_DEBUG=1
export DD_TRACE_LOG_FILE=/tmp/di.log
php app.php
```

**Look for:** Probe registration, snapshot collection events.

### 5. Production Error Monitoring

```bash
export DD_TRACE_LOG_LEVEL=error
export DD_TRACE_LOG_FILE=/var/log/datadog-errors.log
export DD_TRACE_STARTUP_LOGS=1
php-fpm
```

**Look for:** Errors only, with startup JSON for configuration audit.

---

## Limitations and Gotchas

### No Per-Component Configuration

**Limitation:** PHP uses a global log level; cannot isolate specific components (e.g., "only DI logs").

**Workaround:** Use log level filtering:
- `error` - Only errors
- `warn` - Errors + warnings
- `info` - Errors + warnings + info (includes startup)
- `debug` - Errors + warnings + info + debug (includes spans)
- `trace` - Everything (includes hooks)

### CLI vs Non-CLI SAPIs

**CLI:**
- No runtime INI changes
- Startup logs demoted to error level when debug enabled
- Requires restart for log level changes

**Non-CLI (FPM, Apache):**
- Runtime INI changes supported
- Full startup logs at info level
- Can change log level per request

### No Full Span Logging

**Limitation:** PHP does not log full span details (tags, metrics, resource, etc.) by default.

**Reason:** Performance - minimize logging overhead in production.

**Alternative:** Use Datadog APM UI to inspect trace data.

### Startup Logs Can Be Verbose

**Issue:** Startup JSON can be ~500-1000 characters.

**Solutions:**
- Disable: `export DD_TRACE_STARTUP_LOGS=0`
- Keep enabled (recommended): Logs only once per process, useful for configuration auditing

---

## Code References

### Configuration
- `/ext/configuration.h:127` - DD_TRACE_DEBUG
- `/ext/configuration.h:234` - DD_TRACE_LOG_LEVEL
- `/ext/configuration.h:236` - DD_TRACE_LOG_FILE
- `/ext/configuration.h:203` - DD_TRACE_STARTUP_LOGS
- `/ext/configuration.h:207` - DD_TRACE_ONCE_LOGS
- `/ext/configuration.h:163` - DD_TRACE_RATE_LIMIT
- `/ext/configuration.h:253-256` - DI configuration

### C Logging
- `/ext/logging.c:228-234` - `ddtrace_alter_dd_trace_debug()`
- `/ext/logging.c:124-165` - Log file rotation
- `/ext/logging.h:22-25` - Background sender logging

### Span Logging
- `/ext/span.c:54-980` - Span lifecycle logging
- `/ext/span.c:313-319` - Root span start example

### Rate Limiting
- `/ext/limiter/limiter.c:63-80` - Sliding window algorithm
- `/ext/exception_serialize.c:386` - Exception hash limiter

### Startup Logging
- `/ext/startup_logging.c:128-190` - JSON config dump

### PHP Userland
- `/src/api/Log/Logger.php:33-37` - Logger initialization
- `/src/api/Log/DatadogLogger.php:131-175` - JSON logging
- `/src/api/Log/DatadogLogger.php:160-175` - Log injection

### DI / Live Debugger
- `/ext/live_debugger.c`, `/ext/live_debugger.h` - DI implementation
- `/ext/ddtrace.h` - Rust FFI callbacks

### Rust Components
- `/components/log/log.h:14-15` - LOG() macro
- `/components-rs/common.h` - Log level enum

---

## Summary

**dd-trace-php logging is designed for production performance:**

✅ **Lightweight** - Only trace_id/span_id logged, not full span data  
✅ **Startup auditing** - JSON config dump once per process  
✅ **Automatic rotation** - 60s atomic file descriptor rotation  
✅ **Rate limiting** - 100 traces/sec default, shared memory coordination  
✅ **Runtime changes** - INI adjustments for non-CLI SAPIs  
✅ **JSON support** - Userland API for structured logging  
✅ **Component-based** - Internal categorization for future extensibility  

❌ **Global log level** - Cannot isolate specific components  
❌ **No full span logging** - Use APM UI for span inspection  
❌ **CLI limitations** - No runtime INI changes in CLI mode  

**When to use PHP tracer logging:**
- ✅ Startup configuration auditing
- ✅ Trace sampling investigation
- ✅ Agent communication debugging
- ✅ DI probe troubleshooting
- ✅ Production error monitoring

**When NOT to use:**
- ❌ Inspecting span tags/metrics (use APM UI)
- ❌ High-frequency debug logging (minimal output by design)
- ❌ Component-specific filtering (global level only)

---

## Quick Reference

### Enable Debug
```bash
export DD_TRACE_DEBUG=1
```

### Set Log Level
```bash
export DD_TRACE_LOG_LEVEL=debug  # error, warn, info, debug, trace
```

### Configure Log File
```bash
export DD_TRACE_LOG_FILE=/var/log/datadog.log
```

### Enable DI
```bash
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=1
```

### Rate Limiting
```bash
export DD_TRACE_RATE_LIMIT=100  # traces per second
```

### Production Config
```bash
export DD_TRACE_LOG_LEVEL=error
export DD_TRACE_STARTUP_LOGS=1
export DD_TRACE_LOG_FILE=/var/log/datadog-php.log
export DD_LOGS_INJECTION=1
```
