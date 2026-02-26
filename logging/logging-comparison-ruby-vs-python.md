# Datadog Tracer Logging: Ruby vs Python Comparison

## Overview

This document compares logging configuration and behavior between dd-trace-rb (Ruby) and dd-trace-py (Python) tracers, highlighting key differences in how to control tracer and Dynamic Instrumentation (DI) logging.

---

## Major Differences Summary

| Feature | Ruby (dd-trace-rb) | Python (dd-trace-py) |
|---------|-------------------|----------------------|
| **Span detail logging** | ✅ Yes - Full `pretty_inspect` output (~50-150 lines per trace) | ❌ No - Only operational messages |
| **DD_TRACE_DEBUG behavior** | Sets DEBUG + enables verbose span output | Sets DEBUG only (no verbose spans) |
| **Component control** | Programmatic configuration only | ✅ **Environment variables**: `_DD_<COMPONENT>_LOG_LEVEL` |
| **Hierarchical filtering** | Not built-in | ✅ **Prefix trie matching** system |
| **Rate limiting** | No | ✅ Yes (default: 1 log per 60s) |
| **DI verbose logging** | `trace_logging` setting | `_DD_DEBUGGING_LOG_LEVEL=DEBUG` |
| **Configuration approach** | Ruby code blocks | Environment variables |
| **Logger output destination** | Shared logger with entire app | Separate `ddtrace.*` logger hierarchy |
| **Caller stack traces** | In debug mode for all logs | In debug mode for all logs |

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
 Tags: [
   http.method => GET,
   http.url => /api/users,
   http.status_code => 200,
   component => rack,
   span.kind => server
 ]
 Metrics: [
   _dd.top_level => 1.0,
   _sampling_priority_v1 => 1.0
 ]
 Metastruct: []
,
 ... (all spans in trace)
]
```

**Verbosity:** ~50-150 lines per trace depending on span count

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
DEBUG [ddtrace._trace.processor] [__init__.py:353] Starting span: <Span>, trace has 1 spans in the span aggregator
```

**Verbosity:** ~2-5 lines per trace (operational messages only)

---

## Component-Specific Logging Control

### Ruby (dd-trace-rb)

**Method:** Programmatic configuration in Ruby code

```ruby
Datadog.configure do |c|
  # Enable tracer debug, disable DI verbose logging
  c.diagnostics.debug = true
  c.dynamic_instrumentation.internal.trace_logging = false
end
```

**Limitations:**
- Requires code changes
- No built-in environment variable filtering
- Must use custom logger wrapper for component filtering

**Example custom filter:**
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true

  original_logger = c.logger.instance || Datadog::Core::Logger.new($stderr)

  filtered_logger = Object.new.tap do |logger|
    define_singleton_method(:add) do |severity, message = nil, progname = nil, &block|
      msg = message || (block && block.call) || progname

      # Filter out DI logs
      return if msg.to_s.include?('di:')

      original_logger.add(severity, message, progname, &block)
    end

    define_singleton_method(:debug?) { original_logger.debug? }
  end

  c.logger.instance = filtered_logger
end
```

---

### Python (dd-trace-py)

**Method:** Environment variables with hierarchical matching

```bash
# Enable debug for specific component hierarchy
export _DD_DEBUGGING_LOG_LEVEL=DEBUG          # DI only
export _DD_TRACE_PROCESSOR_LOG_LEVEL=DEBUG    # Trace processor only
export _DD_APPSEC_LOG_LEVEL=DEBUG             # AppSec only
export _DD_PROFILING_LOG_LEVEL=DEBUG          # Profiling only

python app.py
```

**How it works:**
1. Environment variables follow pattern: `_DD_<COMPONENT>_LOG_LEVEL=<LEVEL>`
2. Component name converts underscores to dots: `DEBUGGING` → `ddtrace.debugging`
3. Applies to entire logger hierarchy: `ddtrace.debugging.signal`, `ddtrace.debugging.probe`, etc.
4. Uses prefix trie for efficient matching

**Advantages:**
- No code changes required
- Can be set per-environment
- Hierarchical inheritance
- Easy to debug specific components

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
python app.py
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
python app.py
```

---

### Scenario 3: Debug Specific Component Only

#### Ruby
```ruby
# Requires custom logger filter (see example above)
# No built-in component-specific env vars
```

#### Python
```bash
# Easy with environment variables
export _DD_TRACE_PROCESSOR_LOG_LEVEL=DEBUG
python app.py
```

---

## Verbosity Analysis

### Ruby Tracer with DD_TRACE_DEBUG=true

**Per request output (3 spans):**
```
~100-150 lines total:
  - Header: "Writing 3 spans (enabled: true)"
  - Span 1: ~30-40 lines (full details)
  - Span 2: ~30-40 lines (full details)
  - Span 3: ~30-40 lines (full details)
  - Additional operational logs: ~10-20 lines
```

**Components contributing to verbosity:**
1. **Tracing**: Very high (span pretty_inspect)
2. **Profiling**: High (startup diagnostics, ~20-30 lines)
3. **Core/Remote Config**: Medium (client operations, ~5-10 lines)
4. **AppSec**: Medium (security engine, ~3-5 lines)
5. **DI**: Variable (if trace_logging enabled, ~1-10 lines per probe hit)
6. **Data Streams**: Low (~1 line)

**Total per request:** ~200-500+ lines

---

### Python Tracer with DD_TRACE_DEBUG=1

**Per request output (3 spans):**
```
~10-20 lines total:
  - Span finishing: 3 lines (1 per span)
  - Processor messages: 3 lines (1 per span)
  - Context operations: ~4-10 lines
  - No full span details
```

**Components contributing to verbosity:**
1. **Tracing**: Low (operational messages only)
2. **Profiling**: Medium (flush events)
3. **Core**: Low (minimal operational logs)
4. **AppSec**: Low-Medium (WAF operations when active)
5. **DI**: Low-Medium (probe lifecycle only)
6. **All rate-limited** except when at DEBUG level

**Total per request:** ~10-20 lines

---

## Rate Limiting

### Ruby (dd-trace-rb)

**Behavior:**
- ❌ No built-in rate limiting
- All debug logs output immediately
- Can cause log flooding in high-throughput scenarios

**Workaround:**
- Must implement custom logger wrapper
- No standard configuration option

---

### Python (dd-trace-py)

**Behavior:**
- ✅ Built-in rate limiting
- Default: 1 log per 60 seconds per unique location (file:line)
- Automatically disabled when logger at DEBUG level

**Configuration:**
```bash
# Disable rate limiting
export DD_TRACE_LOGGING_RATE=0

# Increase to 5 minutes
export DD_TRACE_LOGGING_RATE=300

# Default 60 seconds
export DD_TRACE_LOGGING_RATE=60
```

**Rate-limited output format:**
```
Original message [5 skipped]
```

---

## Logging Architecture

### Ruby (dd-trace-rb)

**Logger hierarchy:**
```
Root Logger
└── ddtrace (Datadog::Core::Logger)
    ├── Uses same logger for all components
    ├── All components share logger level
    └── Filtering requires custom wrapper
```

**Characteristics:**
- Single logger instance
- All components log to same destination
- Level set globally or via custom filter
- Span output triggered by `diagnostics.debug`

---

### Python (dd-trace-py)

**Logger hierarchy:**
```
Root Logger
└── ddtrace (logging.Logger)
    ├── ddtrace._trace
    │   ├── ddtrace._trace.tracer
    │   ├── ddtrace._trace.span
    │   └── ddtrace._trace.processor
    ├── ddtrace.debugging
    │   ├── ddtrace.debugging._debugger
    │   ├── ddtrace.debugging._signal
    │   └── ddtrace.debugging._probe
    ├── ddtrace.appsec
    ├── ddtrace.profiling
    └── ...
```

**Characteristics:**
- Standard Python logging hierarchy
- Each module has own logger
- Hierarchical level inheritance
- Environment variable control via trie matching
- No verbose span output by design

---

## When to Use Each Approach

### Use Ruby's Verbose Logging When:

✅ Debugging trace data issues
✅ Need to see exact span tags/metrics being sent
✅ Investigating distributed tracing propagation
✅ Understanding span relationships
✅ Development/debugging environments only

⚠️ **Warning:** Very verbose in production

---

### Use Python's Operational Logging When:

✅ Production debugging (minimal noise)
✅ High-throughput applications
✅ Need component-specific debugging
✅ Want to avoid log flooding
✅ Investigating tracer behavior (not span data)

---

## Best Practices

### Ruby (dd-trace-rb)

**Development:**
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true  # Full verbosity
end
```

**Production (Avoid):**
```ruby
# DO NOT enable in production - too verbose
# Use targeted profiling/DI instead
```

**Component isolation:**
```ruby
# Requires custom logger filter
# See full example in dtr-logging.md
```

---

### Python (dd-trace-py)

**Development:**
```bash
export DD_TRACE_DEBUG=1
export DD_TRACE_LOGGING_RATE=0  # Disable rate limit
```

**Production:**
```bash
export DD_TRACE_LOG_LEVEL=WARNING
# Or debug specific component:
export _DD_DEBUGGING_LOG_LEVEL=DEBUG
```

**Component-specific debugging:**
```bash
# Safe in production - targeted and rate-limited
export _DD_TRACE_PROCESSOR_LOG_LEVEL=DEBUG
export _DD_DEBUGGING_LOG_LEVEL=DEBUG
```

---

## Key Takeaways

### Ruby Strengths
- 🔍 **Detailed span inspection** - Full visibility into span data
- 📊 **Complete trace data** - See exactly what's sent to agent
- 🐛 **Rich debugging** - Great for understanding trace propagation

### Ruby Weaknesses
- 📈 **Very verbose** - Hundreds of lines per request
- 🚫 **No component filtering** - All-or-nothing without custom code
- ⚠️ **Production risk** - Can flood logs in high-traffic scenarios

### Python Strengths
- ⚡ **Performance-focused** - Minimal logging overhead
- 🎯 **Granular control** - Component-specific via env vars
- 🔇 **Rate limiting** - Prevents log flooding automatically
- 🏗️ **Hierarchical** - Clean logger inheritance

### Python Weaknesses
- 🔎 **No span details** - Can't see full span data in logs
- 📋 **Operational only** - Must use other tools to inspect trace data
- 🤷 **Different approach** - Requires understanding hierarchy

---

## Migration Guide

### Moving from Ruby to Python Tracer

**Ruby code:**
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true
  c.dynamic_instrumentation.internal.trace_logging = false
end
```

**Python equivalent:**
```bash
export DD_TRACE_DEBUG=1
export _DD_DEBUGGING_LOG_LEVEL=WARNING
```

**Key differences to note:**
1. No full span logging in Python - use APM UI instead
2. Environment variables instead of code configuration
3. Rate limiting active by default
4. Component paths use underscores → dots conversion

---

## References

### Ruby Documentation
- `dtr-logging.md` - Complete Ruby logging guide
- `dtr-component-logging.md` - Component-specific logging

### Python Documentation
- `dd-trace-py-logging.md` - Complete Python logging guide

### Code Locations

**Ruby:**
- `/real.home/claude-5/dtr/lib/datadog/tracing/tracer.rb:566-567` - Span logging
- `/real.home/claude-5/dtr/lib/datadog/core/logger.rb` - Logger with stack traces
- `/real.home/claude-5/dtr/lib/datadog/di/logger.rb` - DI logger wrapper

**Python:**
- `/real.home/claude-5/dd-trace-py/ddtrace/_logger.py` - Main logging config
- `/real.home/claude-5/dd-trace-py/ddtrace/internal/logger.py` - Hierarchical logger
- `/real.home/claude-5/dd-trace-py/ddtrace/_trace/tracer.py` - Tracer logging
