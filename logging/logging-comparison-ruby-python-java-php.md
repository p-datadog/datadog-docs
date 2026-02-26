# Datadog Tracer Logging: Ruby vs Python vs Java vs PHP Comparison

## Overview

This document compares logging configuration and behavior across dd-trace-rb (Ruby), dd-trace-py (Python), dd-trace-java (Java), and dd-trace-php (PHP) tracers, highlighting key differences in how to control tracer and Dynamic Instrumentation (DI) logging.

---

## Major Differences Summary

| Feature | Ruby | Python | Java | PHP |
|---------|------|--------|------|-----|
| **Span detail logging** | ✅ Full `pretty_inspect` (~50-150 lines) | ❌ Operational only | ⚠️ Optional (TraceStructureWriter) | ⚠️ Trace ID + Span ID only |
| **DD_TRACE_DEBUG behavior** | DEBUG + verbose spans | DEBUG only | DEBUG + dynamic switch | DEBUG + startup JSON |
| **Component control** | Programmatic only | `_DD_<COMPONENT>_LOG_LEVEL` | `datadog.slf4j.simpleLogger.log.<package>` | Component-based LOG() macro |
| **Hierarchical filtering** | ❌ Not built-in | ✅ Prefix trie | ✅ Package hierarchy | ✅ Component bitfield |
| **Rate limiting** | ❌ No | ✅ Yes (60s default) | ⚠️ Yes (DI only) | ✅ Yes (trace-level, 100/s) |
| **DI verbose logging** | `trace_logging` setting | `_DD_DEBUGGING_LOG_LEVEL=DEBUG` | `log.com.datadog.debugger=debug` | `DD_DYNAMIC_INSTRUMENTATION_ENABLED=1` |
| **Configuration approach** | Ruby code blocks | Environment variables | System properties | Environment variables |
| **Logger framework** | Custom (`Core::Logger`) | Python `logging` | Custom SLF4J | C extension + Rust FFI |
| **JSON output** | ❌ No | ❌ No | ✅ Yes | ✅ Yes (`DatadogLogger`) |
| **Dynamic level switching** | ❌ No | ❌ No | ✅ Yes (Remote Config) | ⚠️ Yes (INI runtime) |
| **Caller stack traces** | ✅ Debug mode | ✅ Debug mode | ❌ No automatic | ⚠️ File rotation timestamps |
| **Log file rotation** | ❌ No | ❌ No | ❌ No | ✅ Yes (atomic, 60s) |
| **Language** | Pure Ruby | Pure Python | Pure Java | C extension + PHP userland |

---

## DD_TRACE_DEBUG Behavior Comparison

### Ruby (dd-trace-rb)

```bash
export DD_TRACE_DEBUG=true
ruby app.rb
```

**What happens:**
1. Sets logger level to `DEBUG`
2. **Enables full span output** via `pretty_inspect`
3. Logs complete span details for every trace
4. Enables DI `trace_logging` (verbose probe execution logs)
5. Adds caller location `(file.rb:line)` to all logs

**Example output:**
```
[datadog] (lib/datadog/tracing/tracer.rb:567) Writing 3 spans (enabled: true)
[
 Name: rack.request
 Span ID: 1234567890123456789
 Parent ID: 0
 Trace ID: 9876543210987654321
 Type: web
 Service: my-rails-app
 Resource: GET /api/users
 Error: 0
 Start: 1708875123456789000
 End: 1708875123567890000
 Duration: 0.111101
 Tags: [...]
 Metrics: [...]
]
```

**Verbosity:** ~50-150 lines per trace

---

### Python (dd-trace-py)

```bash
export DD_TRACE_DEBUG=1
python app.py
```

**What happens:**
1. Sets logger level to `DEBUG`
2. **Does NOT enable verbose span output** (by design)
3. Logs operational messages only
4. Disables rate limiting
5. Enables file output when `DD_TRACE_LOG_FILE` is set

**Example output:**
```
DEBUG [ddtrace._trace.tracer] [tracer.py:588] finishing span - <Span(name='flask.request', ...)> (enabled:True)
DEBUG [ddtrace._trace.processor] [__init__.py:353] Starting span: <Span>, trace has 1 spans
```

**Verbosity:** ~2-5 lines per trace

---

### Java (dd-trace-java)

```bash
export DD_TRACE_DEBUG=true
# OR
java -Ddd.trace.debug=true -javaagent:dd-java-agent.jar -jar app.jar
```

**What happens:**
1. Sets logger level to `DEBUG`
2. **Does NOT enable verbose span output by default**
3. Logs operational messages (span finish, trace writes)
4. Can enable `TraceStructureWriter` for detailed output
5. Supports JSON format output
6. **Can be dynamically switched via Remote Config**

**Example output:**
```
[dd.trace 2024-02-25 12:00:00:123 -0500] [DEBUG] [PendingTrace] [Thread-1] - t_id=1234567890123456789 -> wrote partial trace of size 3
[dd.trace 2024-02-25 12:00:00:124 -0500] [DEBUG] [DDSpanContext] [Thread-1] - Span finished: operation=servlet.request
```

**With TraceStructureWriter enabled:**
```
TRACE_ID: 1234567890123456789
  SPAN_ID: 9876543210987654321 (Parent: 0)
    Operation: servlet.request
    Service: my-java-app
    Resource: GET /api/users
    Tags: {http.method=GET, http.url=/api/users, ...}
```

**Verbosity:** ~2-10 lines per trace (50-100 with TraceStructureWriter)

---

### PHP (dd-trace-php)

```bash
export DD_TRACE_DEBUG=1
# OR
DD_TRACE_DEBUG=1 php app.php
```

**What happens:**
1. Sets logger level to `DEBUG` (enables TRACE, DEBUG, INFO, WARN, ERROR)
2. **Logs startup configuration as JSON** (once per process)
3. Logs span lifecycle with trace_id/span_id
4. **Does NOT log full span details** (tags, metrics, etc.)
5. Enables file output when `DD_TRACE_LOG_FILE` is set
6. Automatic log file rotation (60s)
7. Can switch log level at runtime via INI

**Startup output (once):**
```
DATADOG TRACER CONFIGURATION - {"date":"2024-02-25T10:30:45Z","os_name":"Linux","os_version":"6.14.0","version":"1.3.0","lang":"php","lang_version":"8.3.0","env":"production","enabled":true,"service":"my-php-app","agent_url":"http://localhost:8126","debug":true,"analytics_enabled":false,"sample_rate":1.000000,"sampling_rules":"[]","tags":{},"service_mapping":{},...}
```

**Span lifecycle output:**
```
[dd-Mon-2024 10:30:45 UTC] [SPAN_TRACE] Starting new root span: trace_id=1234567890123456789, span_id=9876543210987654321, parent_id=0, SpanStack=1, parent_SpanStack=0
[dd-Mon-2024 10:30:45 UTC] [SPAN_TRACE] Closing span: trace_id=1234567890123456789, span_id=9876543210987654321
```

**Verbosity:** ~3-8 lines per trace (+ one-time startup JSON ~500-1000 chars)

---

## Component-Specific Logging Control

### Ruby (dd-trace-rb)

**Method:** Programmatic configuration

```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true
  c.dynamic_instrumentation.internal.trace_logging = false

  # Requires custom logger filter for component isolation
  original_logger = c.logger.instance || Datadog::Core::Logger.new($stderr)
  filtered_logger = CustomFilter.new(original_logger)
  c.logger.instance = filtered_logger
end
```

**Limitations:**
- Requires code changes
- No built-in environment variable filtering
- Must implement custom wrapper

---

### Python (dd-trace-py)

**Method:** Environment variables with prefix trie

```bash
# Enable debug for specific component hierarchy
export _DD_DEBUGGING_LOG_LEVEL=DEBUG          # DI only
export _DD_TRACE_PROCESSOR_LOG_LEVEL=DEBUG    # Trace processor
export _DD_APPSEC_LOG_LEVEL=DEBUG             # AppSec
export _DD_PROFILING_LOG_LEVEL=DEBUG          # Profiling
```

**How it works:**
- Pattern: `_DD_<COMPONENT>_LOG_LEVEL=<LEVEL>`
- Underscores convert to dots: `DEBUGGING` → `ddtrace.debugging`
- Hierarchical inheritance (applies to all submodules)

---

### Java (dd-trace-java)

**Method:** System properties with package hierarchy

```bash
# Enable debug for specific packages
java -Ddd.trace.debug=false \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=debug \
     -Ddatadog.slf4j.simpleLogger.log.datadog.trace.core=debug \
     -Ddatadog.slf4j.simpleLogger.log.datadog.remoteconfig=debug \
     -javaagent:dd-java-agent.jar -jar app.jar
```

**How it works:**
- Pattern: `datadog.slf4j.simpleLogger.log.<package.name>=<level>`
- Walks up package hierarchy to find first match
- Example: `com.datadog.debugger.instrumentation.Foo` checks:
  1. `...log.com.datadog.debugger.instrumentation.Foo`
  2. `...log.com.datadog.debugger.instrumentation`
  3. `...log.com.datadog.debugger`
  4. `...log.com.datadog`
  5. Falls back to default

---

### PHP (dd-trace-php)

**Method:** Component-based bitfield + log level filtering

**Environment variables:**
```bash
# Global log level
export DD_TRACE_LOG_LEVEL=debug          # error, warn, info, debug, trace

# Startup log control (JSON config dump)
export DD_TRACE_STARTUP_LOGS=1           # Enable startup logs (default: true)
export DD_TRACE_ONCE_LOGS=1              # Log once-only messages (default: true)

# Component-specific (experimental)
# Note: PHP uses component bitfield internally, not user-configurable per component
```

**How it works:**
- Log level filter: Global `DD_TRACE_LOG_LEVEL` applies to all components
- Component categories (internal):
  - `SPAN` (component ID: 3, levels: DEBUG, TRACE)
  - `STARTUP` (component ID: 2, levels: INFO, WARN)
  - `HOOK` (component ID: 4, level: TRACE)
  - `DEFAULT` (component ID: 0, all levels)
- LOG() macro checks: `if (ddog_shall_log(DDOG_LOG_##source))`
- Bitfield format: `level | (component_id << 4) | flags`

**Runtime INI changes:**
```php
// Can change at runtime (limited to non-CLI SAPIs)
ini_set('datadog.trace.debug', '1');
ini_set('datadog.trace.log_level', 'trace');
```

**Component isolation not directly configurable** - would require code modification or custom log processor

---

## Configuration Scenarios

### Scenario 1: Only Tracer Logs (No DI)

#### Ruby
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true
  c.dynamic_instrumentation.internal.trace_logging = false
end
```

#### Python
```bash
export DD_TRACE_DEBUG=1
export _DD_DEBUGGING_LOG_LEVEL=WARNING
```

#### Java
```bash
java -Ddd.trace.debug=true \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=warn \
     -javaagent:dd-java-agent.jar -jar app.jar
```

#### PHP
```bash
# Enable tracer debug, disable DI
export DD_TRACE_DEBUG=1
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=0
php app.php
```

---

### Scenario 2: Only DI Logs (No Tracer)

#### Ruby
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = false
  c.logger.level = ::Logger::DEBUG
  c.dynamic_instrumentation.internal.trace_logging = true
end
```

#### Python
```bash
export _DD_DEBUGGING_LOG_LEVEL=DEBUG
export DD_TRACE_LOG_LEVEL=INFO
```

#### Java
```bash
java -Ddatadog.slf4j.simpleLogger.defaultLogLevel=info \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=debug \
     -javaagent:dd-java-agent.jar -jar app.jar
```

#### PHP
```bash
# Enable DI, keep tracer at higher log level
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=1
export DD_TRACE_LOG_LEVEL=warn  # Only warn/error for tracer
php app.php
```

**Note:** PHP doesn't have separate DI log level control; DI logs go through same logger

---

### Scenario 3: Debug Specific Component

#### Ruby
```ruby
# Requires custom logger filter
# No built-in component-specific env vars
```

#### Python
```bash
export _DD_TRACE_PROCESSOR_LOG_LEVEL=DEBUG
```

#### Java
```bash
java -Ddatadog.slf4j.simpleLogger.log.datadog.trace.core.processor=debug \
     -javaagent:dd-java-agent.jar -jar app.jar
```

#### PHP
```bash
# No per-component configuration
# Global log level only:
export DD_TRACE_LOG_LEVEL=debug

# Can disable specific features entirely:
export DD_TRACE_STARTUP_LOGS=0          # No startup JSON
export DD_TRACE_ONCE_LOGS=0             # Allow repeated logs
```

---

### Scenario 4: Production Debugging (Minimal Noise)

#### Ruby
```ruby
# Not recommended - too verbose
# Use APM UI or targeted profiling instead
```

#### Python
```bash
export DD_TRACE_LOG_LEVEL=WARNING
# Or component-specific:
export _DD_DEBUGGING_LOG_LEVEL=DEBUG
```

#### Java
```bash
java -Ddatadog.slf4j.simpleLogger.defaultLogLevel=warn \
     -Ddd.log.format.json=true \
     -javaagent:dd-java-agent.jar -jar app.jar
```

#### PHP
```bash
# Safe production config
export DD_TRACE_LOG_LEVEL=error
export DD_TRACE_STARTUP_LOGS=1          # Keep startup JSON (once only)
export DD_TRACE_LOG_FILE=/var/log/datadog-php.log
php app.php
```

---

## Verbosity Analysis

### Ruby with DD_TRACE_DEBUG=true

**Per request (3 spans):**
- Header: "Writing 3 spans..."
- Span 1: ~30-40 lines (full details)
- Span 2: ~30-40 lines (full details)
- Span 3: ~30-40 lines (full details)
- Operational: ~10-20 lines

**Total:** ~100-150 lines per trace

**Components:**
1. **Tracing**: Very high (span pretty_inspect)
2. **Profiling**: High (~20-30 lines startup)
3. **Core/Remote Config**: Medium (~5-10 lines)
4. **AppSec**: Medium (~3-5 lines)
5. **DI**: Variable (~1-10 per probe)

---

### Python with DD_TRACE_DEBUG=1

**Per request (3 spans):**
- Span finishing: 3 lines
- Processor messages: 3 lines
- Context operations: ~4-10 lines

**Total:** ~10-20 lines per trace

**Note:** Rate-limited except at DEBUG level

---

### Java with dd.trace.debug=true

**Per request (3 spans):**
- Partial trace writes: 1-2 lines
- Span finished: 3 lines
- Context operations: ~2-5 lines

**Total:** ~5-10 lines per trace (without TraceStructureWriter)

**With TraceStructureWriter:** ~50-100 lines per trace

**Note:** DI probes rate-limited (100/sec snapshots, 5000/sec logs)

---

### PHP with DD_TRACE_DEBUG=1

**First request (includes startup):**
- Startup JSON: ~500-1000 chars (1 line, once per process)
- Span lifecycle: ~3-8 lines per trace
  - Root span start: 1 line (trace_id, span_id, parent_id)
  - Child span starts: ~1-2 lines
  - Span closes: ~1-3 lines

**Subsequent requests:**
- Span lifecycle only: ~3-8 lines per trace

**Total:** ~3-8 lines per trace (+ one-time startup)

**Components:**
1. **SPAN_TRACE**: Medium (trace_id, span_id logging)
2. **STARTUP**: Low (once-only JSON config)
3. **HOOK_TRACE**: Low (instrumentation hooks at TRACE level)
4. **ERROR/WARN**: Variable (excluded modules, telemetry warnings)

---

## Rate Limiting

### Ruby
- ❌ No built-in rate limiting
- All debug logs output immediately
- Can cause log flooding

---

### Python
- ✅ Built-in rate limiting
- Default: 1 log per 60s per location
- Disabled at DEBUG level
- Configurable via `DD_TRACE_LOGGING_RATE`

```bash
export DD_TRACE_LOGGING_RATE=0    # Disable
export DD_TRACE_LOGGING_RATE=300  # 5 minutes
```

---

### Java
- ✅ DI probe rate limiting only
- Snapshot rate: 1/sec per probe (100/sec global)
- Log rate: 5000/sec global
- Tracer logs: No rate limiting
- Adaptive sampling based on time windows

---

### PHP
- ✅ Trace-level rate limiting
- Default: 100 traces/second (configurable via `DD_TRACE_RATE_LIMIT`)
- **Mechanism:** Shared memory sliding window counter
  - Atomic operations across forked processes
  - Nanosecond precision (`zend_hrtime()`)
  - Anonymous shared memory for inter-process coordination
- **Exception hash rate limiting:** Prevents duplicate exception captures
- **LOG_ONCE flag:** Messages with `LOG_ONCE` only output once per process
- **No per-log-statement rate limiting** (unlike Python's 60s per location)

**Configuration:**
```bash
export DD_TRACE_RATE_LIMIT=100    # Max 100 traces/sec
export DD_TRACE_ONCE_LOGS=1       # Enable once-only logs (default: true)
```

---

## Special Features

### Ruby

**Strengths:**
- 🔍 **Complete span introspection** - See all span data
- 📊 **Ruby object inspection** - Full `pretty_inspect` output
- 🐛 **Rich debugging** - Understand trace propagation

**Limitations:**
- 📈 Very verbose
- 🚫 No component filtering
- ⚠️ Production risk (log flooding)

---

### Python

**Strengths:**
- ⚡ **Performance-focused** - Minimal overhead
- 🎯 **Granular control** - Component-specific env vars
- 🔇 **Rate limiting** - Prevents flooding
- 🏗️ **Hierarchical** - Clean logger inheritance

**Limitations:**
- 🔎 No span details
- 📋 Operational only
- 🤷 Different approach

---

### Java

**Strengths:**
- 🔄 **Dynamic switching** - Runtime level changes via Remote Config
- 📦 **SLF4J compatible** - Standard Java logging
- 📊 **JSON output** - Structured logging support
- 🎛️ **Fine-grained control** - Per-package configuration
- ⚖️ **Balanced** - Optional verbose output

**Limitations:**
- 🔧 Complex configuration (system properties)
- 📝 No automatic span pretty-print
- 🧰 Requires understanding SLF4J

---

### PHP

**Strengths:**
- 🔄 **Automatic log rotation** - 60s atomic file descriptor rotation
- 📊 **JSON startup config** - Complete configuration dump
- ⚡ **Lightweight** - Minimal span logging (IDs only)
- 🔒 **Thread-safe rate limiting** - Shared memory with atomic ops
- 🎯 **Runtime INI changes** - Can adjust log level without restart (non-CLI)
- 📦 **Component-based** - Internal bitfield for log categorization
- 🌐 **Multi-language** - C extension + Rust FFI + PHP userland

**Limitations:**
- 🔧 No per-component user configuration
- 📝 No full span details in logs
- 🧰 Global log level only
- ⚠️ Startup logs can be verbose (but once-only)

---

## JSON Logging Support

### Ruby
❌ Not supported

### Python
❌ Not supported

### Java
✅ **Full support**

```bash
java -Ddd.log.format.json=true -javaagent:dd-java-agent.jar -jar app.jar
```

**Output format:**
```json
{
  "origin": "dd.trace",
  "date": "2024-02-25T12:00:00.123-0500",
  "logger.thread_name": "Thread-1",
  "level": "DEBUG",
  "logger.name": "datadog.trace.core.PendingTrace",
  "message": "t_id=1234567890123456789 -> wrote partial trace of size 3",
  "exception": "..."
}
```

### PHP
✅ **Partial support** (userland only)

**PHP Userland API:**
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

**Extension logs (C side):** Plain text with timestamps
```
[dd-Mon-2024 10:30:45 UTC] [DEBUG] Span finished: span_id=9876543210987654321
```

---

## Log File Rotation

### Ruby
❌ Not supported - manual log rotation required

### Python
❌ Not supported - manual log rotation required

### Java
❌ Not supported - manual log rotation required

### PHP
✅ **Automatic atomic rotation**

**Implementation:** `/ext/logging.c:124-165`
- Rotates file descriptor every 60 seconds
- Atomic operations prevent race conditions
- Reopens file automatically
- Works with custom file paths and PHP streams

**Configuration:**
```bash
export DD_TRACE_LOG_FILE=/var/log/datadog-php.log
```

**Supported destinations:**
- File paths: `/var/log/app.log`
- PHP streams: `php://stderr`, `php://stdout`
- Falls back to PHP `error_log` directive

---

## Dynamic Log Level Switching

### Ruby
❌ Not supported - requires restart

### Python
❌ Not supported - requires restart

### Java
✅ **Full support via Remote Config**

**Implementation:**
- `GlobalLogLevelSwitcher.get().switchLevel(LogLevel.DEBUG)`
- `TracingConfigPoller` monitors remote config changes
- Can switch between DEBUG/INFO/WARN at runtime
- No application restart required

### PHP
⚠️ **Partial support via INI runtime changes**

**Mechanism:**
```php
// Runtime INI change (non-CLI SAPIs only)
ini_set('datadog.trace.debug', '1');
ini_set('datadog.trace.log_level', 'trace');
```

**Limitations:**
- Only works for non-CLI SAPIs (Apache, FPM, etc.)
- CLI requires restart
- No remote config integration
- INI callbacks trigger level adjustments: `ddtrace_alter_dd_trace_debug()`, `ddtrace_alter_dd_trace_log_level()`

---

## Configuration Initialization

### Ruby
1. Check `DD_TRACE_DEBUG` env var
2. Set logger level
3. Configure file output if specified
4. Enable span pretty_inspect if debug

### Python
1. Check `DD_TRACE_DEBUG` env var
2. Check `DD_TRACE_LOG_LEVEL` env var
3. Build logger prefix trie from `_DD_*_LOG_LEVEL` vars
4. Apply rate limiting
5. Configure file output

### Java
1. **Bootstrap phase**: Check system properties/env vars
2. **Priority order**:
   - `dd.trace.debug` → DEBUG
   - `dd.trace.log.level` → specified level
   - `dd.log.level` → specified level
   - `OTEL_LOG_LEVEL` → mapped value
   - Default: INFO or WARN
3. **Component-specific**: Apply per-package levels
4. **Runtime**: Monitor remote config for dynamic changes

### PHP
1. **Extension initialization** (`MINIT`):
   - Parse `DD_TRACE_DEBUG` → `dd_log_set_level(true/false)`
   - Parse `DD_TRACE_LOG_LEVEL` → `ddog_set_log_level(level_str)`
   - Priority: `DD_TRACE_DEBUG` overrides `DD_TRACE_LOG_LEVEL`
   - Configure `DD_TRACE_LOG_FILE` path
2. **SAPI-specific adjustments**:
   - Non-CLI with startup logs: Full debug output
   - CLI or no startup logs: `"debug,startup=error"` (demotes startup to error)
3. **First request** (if `DD_TRACE_STARTUP_LOGS=1`):
   - Log startup JSON configuration dump (once only via `DD_TRACE_ONCE_LOGS`)
4. **Runtime INI changes** (non-CLI only):
   - `ddtrace_alter_dd_trace_debug()` callback
   - `ddtrace_alter_dd_trace_log_level()` callback
5. **Log file rotation**: Every 60s atomic FD rotation

---

## Best Practices

### Ruby

**Development:**
```ruby
Datadog.configure { |c| c.diagnostics.debug = true }
```

**Production:**
```ruby
# Avoid DD_TRACE_DEBUG - too verbose
# Use targeted profiling/DI instead
```

---

### Python

**Development:**
```bash
export DD_TRACE_DEBUG=1
export DD_TRACE_LOGGING_RATE=0
```

**Production:**
```bash
export DD_TRACE_LOG_LEVEL=WARNING
# Or component-specific:
export _DD_DEBUGGING_LOG_LEVEL=DEBUG
```

---

### Java

**Development:**
```bash
java -Ddd.trace.debug=true \
     -Ddd.trace.debug.log=true \
     -javaagent:dd-java-agent.jar -jar app.jar
```

**Production:**
```bash
# Conservative default
java -Ddatadog.slf4j.simpleLogger.defaultLogLevel=warn \
     -javaagent:dd-java-agent.jar -jar app.jar

# Component-specific debugging (safe)
java -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=debug \
     -javaagent:dd-java-agent.jar -jar app.jar
```

**JSON logging (production):**
```bash
java -Ddd.log.format.json=true \
     -Ddatadog.slf4j.simpleLogger.defaultLogLevel=info \
     -javaagent:dd-java-agent.jar -jar app.jar
```

---

### PHP

**Development:**
```bash
# Full debug with all messages
export DD_TRACE_DEBUG=1
export DD_TRACE_LOG_FILE=/tmp/dd-trace-debug.log
export DD_TRACE_STARTUP_LOGS=1
export DD_TRACE_ONCE_LOGS=0  # Allow repeated logs
php app.php
```

**Production:**
```bash
# Conservative - errors only
export DD_TRACE_LOG_LEVEL=error
export DD_TRACE_STARTUP_LOGS=1  # Keep config JSON (once only)
export DD_TRACE_LOG_FILE=/var/log/datadog-php.log
export DD_TRACE_RATE_LIMIT=100  # Limit to 100 traces/sec

# Run with your web server
php-fpm
```

**Temporary debugging (non-CLI):**
```php
// Enable debug for a single request (web servers only)
ini_set('datadog.trace.debug', '1');
// ... debug code here ...
ini_set('datadog.trace.debug', '0');
```

**Background sender debugging:**
```bash
# Verbose CURL output for agent communication
export DD_TRACE_AGENT_DEBUG_VERBOSE_CURL=1
export DD_TRACE_DEBUG_CURL_OUTPUT=1
```

---

## Migration Guide

### Ruby → Python

**Ruby:**
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true
  c.dynamic_instrumentation.internal.trace_logging = false
end
```

**Python:**
```bash
export DD_TRACE_DEBUG=1
export _DD_DEBUGGING_LOG_LEVEL=WARNING
```

**Key differences:**
- No full span logging in Python
- Environment variables vs code
- Rate limiting active by default

---

### Ruby → Java

**Ruby:**
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true
end
```

**Java:**
```bash
java -Ddd.trace.debug=true -javaagent:dd-java-agent.jar -jar app.jar
```

**Key differences:**
- System properties vs Ruby code
- Optional TraceStructureWriter for span details
- JSON output available
- Dynamic level switching supported

---

### Ruby → PHP

**Ruby:**
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true
end
```

**PHP:**
```bash
export DD_TRACE_DEBUG=1
export DD_TRACE_LOG_FILE=/var/log/datadog.log
php app.php
```

**Key differences:**
- Environment variables vs Ruby code
- Startup JSON config dump (once only)
- No full span details, only trace_id/span_id
- Automatic log rotation (60s)
- Rate limiting at trace level (100/sec default)

---

### Python → Java

**Python:**
```bash
export _DD_DEBUGGING_LOG_LEVEL=DEBUG
export DD_TRACE_LOG_LEVEL=INFO
```

**Java:**
```bash
java -Ddatadog.slf4j.simpleLogger.defaultLogLevel=info \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=debug \
     -javaagent:dd-java-agent.jar -jar app.jar
```

**Key differences:**
- Longer property names
- Package hierarchy vs module paths
- Can use JSON format
- Runtime level switching available

---

### Python → PHP

**Python:**
```bash
export DD_TRACE_DEBUG=1
export _DD_DEBUGGING_LOG_LEVEL=WARNING
```

**PHP:**
```bash
export DD_TRACE_DEBUG=1
export DD_TRACE_LOG_LEVEL=info
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=0
```

**Key differences:**
- No component-specific log levels in PHP
- PHP has startup JSON dump
- PHP has automatic log rotation
- PHP rate limits at trace level, not per log statement
- DI logs through same logger in PHP

---

### Java → PHP

**Java:**
```bash
java -Ddd.trace.debug=true \
     -Ddd.log.format.json=true \
     -javaagent:dd-java-agent.jar -jar app.jar
```

**PHP:**
```bash
export DD_TRACE_DEBUG=1
export DD_TRACE_LOG_FILE=/var/log/datadog.log
php app.php
```

**Key differences:**
- PHP JSON logging only for userland API (not extension logs)
- PHP has automatic log rotation, Java doesn't
- PHP startup config logged as JSON once per process
- PHP has simpler configuration (fewer knobs)
- Java has dynamic remote config, PHP has INI runtime changes

---

## Summary Table: Quick Reference

| Capability | Ruby | Python | Java | PHP |
|------------|------|--------|------|-----|
| **Verbose span output** | ✅ Always (debug) | ❌ Never | ⚠️ Optional | ❌ Never (IDs only) |
| **Component filtering** | ❌ Custom code | ✅ Env vars | ✅ System props | ❌ Global only |
| **Rate limiting** | ❌ No | ✅ Yes (all) | ⚠️ Yes (DI only) | ✅ Yes (trace-level) |
| **JSON output** | ❌ No | ❌ No | ✅ Yes | ⚠️ Userland only |
| **Dynamic switching** | ❌ No | ❌ No | ✅ Yes | ⚠️ INI runtime |
| **Log rotation** | ❌ No | ❌ No | ❌ No | ✅ Yes (60s atomic) |
| **Production safe** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes |
| **Easy config** | ⚠️ Code required | ✅ Env vars | ⚠️ System props | ✅ Env vars |
| **Startup config dump** | ❌ No | ❌ No | ❌ No | ✅ Yes (JSON) |
| **Background sender logs** | ❌ No | ❌ No | ❌ No | ✅ Yes (optional) |

---

## Language-Specific Implementation Details

### Ruby
- **Language:** Pure Ruby
- **Logger:** `Datadog::Core::Logger` (custom)
- **Location:** `/dtr/lib/datadog/core/logger.rb`
- **Span logging:** `/dtr/lib/datadog/tracing/tracer.rb:566-567`
- **pretty_inspect:** `/dtr/lib/datadog/tracing/span.rb:172-204`

### Python
- **Language:** Pure Python
- **Logger:** Python `logging` module
- **Location:** `/dd-trace-py/ddtrace/_logger.py`
- **Hierarchical:** `/dd-trace-py/ddtrace/internal/logger.py`
- **Prefix trie:** `LoggerPrefix.build_trie()`

### Java
- **Language:** Pure Java
- **Logger:** Custom SLF4J implementation
- **Location:** `/dd-trace-java/dd-java-agent/agent-logging/`
- **Package hierarchy:** `SimpleLogger.recursivelyComputeLogLevel()`
- **Remote config:** `GlobalLogLevelSwitcher`, `TracingConfigPoller`

### PHP
- **Language:** C extension + Rust FFI + PHP userland
- **Logger:** 
  - C: `/dd-trace-php/ext/logging.c`, `/dd-trace-php/ext/logging.h`
  - Rust: `/dd-trace-php/components/log/`
  - PHP: `/dd-trace-php/src/api/Log/`
- **Configuration:** `/dd-trace-php/ext/configuration.h:127, 234-236`
- **Rate limiting:** `/dd-trace-php/ext/limiter/limiter.c` (shared memory)
- **Startup logging:** `/dd-trace-php/ext/startup_logging.c:128-190`
- **DI:** `/dd-trace-php/ext/live_debugger.c`, `/dd-trace-php/ext/live_debugger.h`

---

## References

### Ruby Documentation
- `dtr-logging.md` - Complete Ruby logging guide
- `dtr-component-logging.md` - Component-specific Ruby logging

### Python Documentation
- `dd-trace-py-logging.md` - Complete Python logging guide

### Java Documentation
- Configuration: `/internal-api/src/main/java/datadog/trace/api/Config.java`
- Logger: `/dd-java-agent/agent-logging/`
- SLF4J: `/dd-java-agent/agent-logging/src/main/java/datadog/trace/logging/`

### PHP Documentation
- Configuration: `/dd-trace-php/ext/configuration.h`
- C Logging: `/dd-trace-php/ext/logging.c`, `/dd-trace-php/ext/logging.h`
- Rust Components: `/dd-trace-php/components/log/`, `/dd-trace-php/components-rs/common.h`
- PHP API: `/dd-trace-php/src/api/Log/DatadogLogger.php`, `/dd-trace-php/src/api/Log/ErrorLogLogger.php`
- Startup: `/dd-trace-php/ext/startup_logging.c`
- Tests: `/dd-trace-php/tests/Integrations/Custom/Autoloaded/StartupLoggingTest.php`

### Code Locations

**Ruby:**
- Span logging: `/dtr/lib/datadog/tracing/tracer.rb:566-567`
- Logger: `/dtr/lib/datadog/core/logger.rb`
- DI logger: `/dtr/lib/datadog/di/logger.rb`

**Python:**
- Main config: `/dd-trace-py/ddtrace/_logger.py`
- Hierarchical: `/dd-trace-py/ddtrace/internal/logger.py`
- Tracer: `/dd-trace-py/ddtrace/_trace/tracer.py`

**Java:**
- Config: `/dd-trace-java/internal-api/src/main/java/datadog/trace/api/Config.java`
- Logger: `/dd-trace-java/dd-java-agent/agent-logging/src/main/java/datadog/trace/logging/`
- Debugger: `/dd-trace-java/dd-java-agent/agent-debugger/`

**PHP:**
- C Logging: `/dd-trace-php/ext/logging.c:228-234`, `/dd-trace-php/ext/logging.h`
- Configuration: `/dd-trace-php/ext/configuration.h:127, 203-207, 234-236, 253-256`
- Span Logging: `/dd-trace-php/ext/span.c:54-980`
- Rate Limiter: `/dd-trace-php/ext/limiter/limiter.c:63-80`
- Startup Logs: `/dd-trace-php/ext/startup_logging.c:128-190`
- PHP Logger API: `/dd-trace-php/src/api/Log/Logger.php:33-37`
- PHP DatadogLogger: `/dd-trace-php/src/api/Log/DatadogLogger.php:131-175`
- Live Debugger: `/dd-trace-php/ext/live_debugger.c`, `/dd-trace-php/ext/live_debugger.h`
- Rust Components: `/dd-trace-php/components/log/log.h:14-15`, `/dd-trace-php/components-rs/common.h`

---

## Key Takeaways

### When to Use Each Tracer's Logging

**Ruby:**
- ✅ Development debugging of span data
- ✅ Understanding distributed tracing propagation
- ✅ Investigating tag/metric issues
- ❌ Production environments (too verbose)

**Python:**
- ✅ Production debugging (minimal noise)
- ✅ High-throughput applications
- ✅ Component-specific troubleshooting
- ✅ Rate-limited environments

**Java:**
- ✅ Enterprise applications requiring SLF4J integration
- ✅ Environments needing JSON structured logs
- ✅ Dynamic debugging without restarts (Remote Config)
- ✅ Fine-grained package-level control

**PHP:**
- ✅ Web applications (Apache, FPM, nginx)
- ✅ CLI scripts with startup diagnostics
- ✅ Environments requiring log rotation
- ✅ Lightweight trace-level logging
- ✅ Startup configuration auditing

---

## Environment Variable Summary

| Variable | Ruby | Python | Java | PHP |
|----------|------|--------|------|-----|
| `DD_TRACE_DEBUG` | ✅ | ✅ | ✅ | ✅ |
| `DD_TRACE_LOG_LEVEL` | ❌ | ✅ | ⚠️ `dd.trace.log.level` | ✅ |
| `DD_TRACE_LOG_FILE` | ✅ | ✅ | ❌ | ✅ |
| `DD_TRACE_STARTUP_LOGS` | ❌ | ❌ | ❌ | ✅ |
| `DD_TRACE_ONCE_LOGS` | ❌ | ❌ | ❌ | ✅ |
| `DD_TRACE_RATE_LIMIT` | ❌ | ⚠️ `DD_TRACE_LOGGING_RATE` | ❌ | ✅ (trace-level) |
| `_DD_<COMPONENT>_LOG_LEVEL` | ❌ | ✅ | ❌ | ❌ |
| `datadog.slf4j.simpleLogger.log.<pkg>` | ❌ | ❌ | ✅ | ❌ |
| `DD_DYNAMIC_INSTRUMENTATION_ENABLED` | ✅ | ✅ | ✅ | ✅ |
| `DD_LOGS_INJECTION` | ✅ | ✅ | ✅ | ✅ (userland) |

---

**Total Documentation Coverage:**
- **4 tracers**: Ruby, Python, Java, PHP
- **12 main sections**: DD_TRACE_DEBUG, component control, scenarios, verbosity, rate limiting, JSON, rotation, dynamic switching, etc.
- **4 migration paths**: Ruby↔Python↔Java↔PHP
- **Complete code references** for all implementations
