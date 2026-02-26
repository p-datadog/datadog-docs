# Datadog Tracer Logging: Ruby vs Python vs Java Comparison

## Overview

This document compares logging configuration and behavior across dd-trace-rb (Ruby), dd-trace-py (Python), and dd-trace-java (Java) tracers, highlighting key differences in how to control tracer and Dynamic Instrumentation (DI) logging.

---

## Major Differences Summary

| Feature | Ruby (dd-trace-rb) | Python (dd-trace-py) | Java (dd-trace-java) |
|---------|-------------------|----------------------|---------------------|
| **Span detail logging** | ✅ Yes - Full `pretty_inspect` (~50-150 lines) | ❌ No - Operational only | ⚠️ Optional - via `TraceStructureWriter` |
| **DD_TRACE_DEBUG behavior** | DEBUG + verbose spans | DEBUG only | DEBUG only + dynamic switching |
| **Component control** | Programmatic only | `_DD_<COMPONENT>_LOG_LEVEL` | `datadog.slf4j.simpleLogger.log.<package>` |
| **Hierarchical filtering** | Not built-in | ✅ Prefix trie | ✅ Package hierarchy |
| **Rate limiting** | No | ✅ Yes (60s default) | ✅ Yes (DI probes only) |
| **DI verbose logging** | `trace_logging` setting | `_DD_DEBUGGING_LOG_LEVEL=DEBUG` | `-Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=debug` |
| **Configuration approach** | Ruby code blocks | Environment variables | System properties / env vars |
| **Logger framework** | Custom (`Core::Logger`) | Python `logging` | Custom SLF4J implementation |
| **JSON output** | No | No | ✅ Yes - `dd.log.format.json=true` |
| **Dynamic level switching** | No | No | ✅ Yes - via Remote Config |
| **Caller stack traces** | ✅ Debug mode all logs | ✅ Debug mode all logs | ❌ No automatic |

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

**Available log levels:**
- `TRACE` - Most verbose
- `DEBUG` - Debug messages
- `INFO` - Informational
- `WARN` - Warnings
- `ERROR` - Errors only
- `OFF` - No logging

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

## Summary Table: Quick Reference

| Capability | Ruby | Python | Java |
|------------|------|--------|------|
| **Verbose span output** | ✅ Always (with debug) | ❌ Never | ⚠️ Optional (TraceStructureWriter) |
| **Component filtering** | ❌ Custom code | ✅ Env vars | ✅ System props |
| **Rate limiting** | ❌ No | ✅ Yes (all) | ⚠️ Yes (DI only) |
| **JSON output** | ❌ No | ❌ No | ✅ Yes |
| **Dynamic switching** | ❌ No | ❌ No | ✅ Yes |
| **Production safe** | ❌ No | ✅ Yes | ✅ Yes |
| **Easy config** | ⚠️ Code required | ✅ Env vars | ⚠️ System props |
| **SLF4J compatible** | N/A | N/A | ✅ Yes |

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
