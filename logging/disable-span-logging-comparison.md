# Disabling Span Logging While Keeping Other Debug Logs: Cross-Tracer Comparison

## Overview

This document compares how to **disable tracer span logging** specifically while keeping debug-level logging for other components (AppSec, Dynamic Instrumentation, Profiling, etc.) across all 7 Datadog tracers.

### Why This Matters

Span logging can be extremely verbose:
- **Ruby:** ~50-150 lines per trace with full span details
- **Go:** ~15-40 lines per trace with all tags/metrics
- **.NET:** ~10-30 lines per trace with complete span objects
- **Java:** ~5-10 lines per trace (50-100 with TraceStructureWriter)

In high-throughput applications, this can:
- Generate gigabytes of logs per day
- Fill disk storage quickly
- Make it difficult to find relevant non-span logs
- Impact performance due to logging overhead

**Common scenario:** You want to debug Dynamic Instrumentation probe behavior or AppSec rule matching, but you don't need to see every span that's created.

---

## Summary Matrix

| Tracer | Can Disable Span Logging? | Method | Ease | Code Changes Required |
|--------|--------------------------|--------|------|----------------------|
| **Ruby** | ⚠️ Partial | Custom logger filter | ⭐ Hard | ✅ Yes |
| **Python** | ✅ N/A | Not logged by default | ⭐⭐⭐⭐⭐ Easy | ❌ No |
| **Java** | ✅ Yes | Per-package properties | ⭐⭐⭐ Moderate | ❌ No |
| **PHP** | ✅ N/A | Not logged by default | ⭐⭐⭐⭐⭐ Easy | ❌ No |
| **.NET** | ⚠️ Partial | Serilog filtering | ⭐⭐ Moderate | ✅ Yes |
| **Go** | ❌ No | Global level only | ⭐ Hard | ✅ Yes |
| **JavaScript** | ✅ Yes | Log level thresholds | ⭐⭐⭐⭐ Easy | ❌ No |

**Key:**
- ✅ Yes = Easy to disable via configuration
- ⚠️ Partial = Possible but requires code changes
- ❌ No = Not possible without modifying tracer code
- N/A = Spans not logged at debug level anyway

---

## Ruby (dd-trace-rb): Custom Filter Required

### Default Behavior

**With DD_TRACE_DEBUG=true:**
```ruby
# Logs FULL span details (~50-150 lines per trace)
[datadog] Writing 3 spans (enabled: true)
[
 Name: rack.request
 Span ID: 1234567890123456789
 Parent ID: 0
 Trace ID: 9876543210987654321
 Type: web
 Service: my-rails-app
 Resource: GET /api/users
 Tags: [http.method => GET, ...]
 Metrics: [_dd.top_level => 1.0, ...]
]
```

**Location:** `/dtr/lib/datadog/tracing/tracer.rb:566-567`

### Solution: Custom Logger Filter

**Implementation:**

```ruby
require 'datadog'

Datadog.configure do |c|
  # Enable debug mode
  c.diagnostics.debug = true
  
  # Get original logger
  original_logger = c.logger.instance || Datadog::Core::Logger.new($stderr)
  
  # Create filtering wrapper
  filtered_logger = Object.new
  
  # Define logging method with span filtering
  filtered_logger.define_singleton_method(:add) do |severity, message = nil, progname = nil, &block|
    msg = message || (block && block.call) || progname
    msg_str = msg.to_s
    
    # FILTER OUT span logging
    # Pattern 1: "Writing N spans"
    return if msg_str.match?(/Writing \d+ spans/)
    
    # Pattern 2: Span pretty_inspect output (array format)
    return if msg_str.start_with?('[') && msg_str.include?('Name:') && msg_str.include?('Span ID:')
    
    # Pattern 3: Individual span details
    return if msg_str.match?(/Span ID:.*Parent ID:.*Trace ID:/)
    
    # Pass through all other logs
    original_logger.add(severity, message, progname, &block)
  end
  
  # Delegate level check methods
  [:debug?, :info?, :warn?, :error?, :fatal?].each do |method|
    filtered_logger.define_singleton_method(method) { original_logger.send(method) }
  end
  
  # Replace logger
  c.logger.instance = filtered_logger
  
  # Keep DI logging if needed
  c.dynamic_instrumentation.internal.trace_logging = true
end
```

### Result

**Logs you WILL see:**
```
[datadog] Profiling enabled
[datadog] AppSec: WAF initialized
[datadog] di: Probe abc123 applied successfully
[datadog] Remote Config: Configuration received
```

**Logs you WON'T see:**
```
[datadog] Writing 3 spans (enabled: true)
[
 Name: rack.request
 ...
]
```

### Pros & Cons

**Pros:**
- ✅ Works without modifying tracer code
- ✅ Can customize filtering logic
- ✅ Can add additional filters for other components

**Cons:**
- ❌ Requires code changes (can't use env vars only)
- ❌ String pattern matching is brittle
- ❌ May break if log format changes
- ❌ More complex than other tracers

### Alternative: Disable Debug Entirely

```ruby
Datadog.configure do |c|
  # Keep logger at INFO
  c.diagnostics.debug = false
  c.logger.level = ::Logger::INFO
  
  # Enable only DI verbose logging
  c.dynamic_instrumentation.internal.trace_logging = true
end
```

**Trade-off:** Loses other debug logs (profiling, remote config, etc.)

---

## Python (dd-trace-py): Not Logged By Default

### Default Behavior

**With DD_TRACE_DEBUG=1:**

Python does **NOT** log full span details by design. You get operational messages only:

```
DEBUG [ddtrace._trace.tracer] finishing span - <Span(name='flask.request', ...)> (enabled:True)
DEBUG [ddtrace._trace.processor] Starting span: <Span>, trace has 1 spans
```

**No verbose span output** - just span lifecycle events (~2-5 lines per trace).

### Solution: Already Solved

**No action needed!** Python's debug logging is already production-safe.

### Disabling Span Lifecycle Logs (Optional)

If you want to remove even the operational span messages:

```bash
export DD_TRACE_LOG_LEVEL=INFO                    # Global at INFO
export _DD_DEBUGGING_LOG_LEVEL=DEBUG              # DI at DEBUG
export _DD_APPSEC_LOG_LEVEL=DEBUG                 # AppSec at DEBUG
export _DD_PROFILING_LOG_LEVEL=DEBUG              # Profiling at DEBUG
export _DD_TRACE_TRACER_LOG_LEVEL=WARNING         # Span lifecycle at WARNING
export _DD_TRACE_SPAN_LOG_LEVEL=WARNING           # Span operations at WARNING
export _DD_TRACE_PROCESSOR_LOG_LEVEL=WARNING      # Processor at WARNING
```

### Result

**Logs you WILL see:**
```
DEBUG [ddtrace.debugging._debugger] Enabling Debugger
DEBUG [ddtrace.appsec._asm] WAF initialized
DEBUG [ddtrace.profiling.collector] Starting collector
```

**Logs you WON'T see:**
```
DEBUG [ddtrace._trace.tracer] finishing span - <Span(...)>
DEBUG [ddtrace._trace.processor] Starting span: <Span>
```

### Pros & Cons

**Pros:**
- ✅ No span details logged by default
- ✅ Operational messages only
- ✅ Environment variable control
- ✅ No code changes needed
- ✅ Production-safe debug logging

**Cons:**
- ⚠️ Can't see full span details even if you want them (use APM UI)

---

## Java (dd-trace-java): Per-Package Control

### Default Behavior

**With dd.trace.debug=true:**

```
[dd.trace 2024-02-25 12:00:00:123] [DEBUG] [DDSpanContext] Span finished: operation=servlet.request
[dd.trace 2024-02-25 12:00:00:124] [DEBUG] [PendingTrace] t_id=1234567890123456789 -> wrote partial trace of size 3
```

Minimal span logging (~5-10 lines per trace). Can enable verbose output with `TraceStructureWriter`.

### Solution: Disable Span Packages

```bash
# Global debug
java -Ddd.trace.debug=true \
     \
     # Disable span-related logging
     -Ddatadog.slf4j.simpleLogger.log.datadog.trace.core.DDSpan=warn \
     -Ddatadog.slf4j.simpleLogger.log.datadog.trace.core.DDSpanContext=warn \
     -Ddatadog.slf4j.simpleLogger.log.datadog.trace.core.PendingTrace=warn \
     -Ddatadog.slf4j.simpleLogger.log.datadog.trace.core.CoreTracer=warn \
     \
     # Keep other components at debug
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=debug \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.appsec=debug \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.profiling=debug \
     \
     -javaagent:dd-java-agent.jar -jar app.jar
```

### Alternative: Concise Configuration

```bash
# Disable entire trace.core package (includes all span logging)
java -Ddd.trace.debug=true \
     -Ddatadog.slf4j.simpleLogger.log.datadog.trace.core=warn \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=debug \
     -javaagent:dd-java-agent.jar -jar app.jar
```

### Result

**Logs you WILL see:**
```
[dd.trace] [DEBUG] [DebuggerAgent] Probe applied: probe_id=abc123
[dd.trace] [DEBUG] [AppSecSystem] WAF rule matched: rule_id=sqli-001
[dd.trace] [DEBUG] [ProfilingController] Starting profiling session
```

**Logs you WON'T see:**
```
[dd.trace] [DEBUG] [DDSpanContext] Span finished: operation=servlet.request
[dd.trace] [DEBUG] [PendingTrace] t_id=1234 -> wrote partial trace of size 3
```

### Disabling TraceStructureWriter

If you previously enabled verbose span output:

```bash
# Ensure TraceStructureWriter is disabled
java -Ddd.trace.debug=true \
     -Ddd.trace.debug.log=false \
     -javaagent:dd-java-agent.jar -jar app.jar
```

### Pros & Cons

**Pros:**
- ✅ Fine-grained package control
- ✅ No code changes needed
- ✅ Standard SLF4J approach
- ✅ Can selectively disable specific span classes

**Cons:**
- ⚠️ Long system property names
- ⚠️ Must know package structure
- ⚠️ Verbose configuration for multiple packages

---

## PHP (dd-trace-php): Not Logged By Default

### Default Behavior

**With DD_TRACE_DEBUG=1:**

PHP logs **minimal span information** - only trace_id and span_id:

```
[dd-Mon-2024 10:30:45 UTC] [SPAN_TRACE] Starting new root span: trace_id=1234, span_id=5678
[dd-Mon-2024 10:30:45 UTC] [SPAN_TRACE] Closing span: trace_id=1234, span_id=5678
```

**No full span details** (~3-8 lines per trace).

### Solution: Already Minimal

**No action needed!** PHP's span logging is already minimal.

### Further Reduction (Optional)

Lower log level to skip SPAN_TRACE logs:

```bash
export DD_TRACE_LOG_LEVEL=debug  # Skip TRACE level (SPAN_TRACE won't log)

# Or completely silence span logs
export DD_TRACE_LOG_LEVEL=info   # Only INFO and higher
```

**Note:** This may also suppress other TRACE-level logs (hooks, etc.)

### Alternative: Disable Startup Logs Only

```bash
export DD_TRACE_DEBUG=1
export DD_TRACE_STARTUP_LOGS=0  # Disable startup JSON (not span logs)
```

### Result

**Logs you WILL see:**
```
[dd-Mon-2024] [DEBUG] DI: Probe registered: probe_id=abc123
[dd-Mon-2024] [DEBUG] Exception captured for replay
```

**Logs you MAY NOT see (if using debug level):**
```
[dd-Mon-2024] [SPAN_TRACE] Starting span: trace_id=1234
[dd-Mon-2024] [SPAN_TRACE] Closing span: trace_id=1234
```

### Pros & Cons

**Pros:**
- ✅ Already minimal (only trace_id/span_id)
- ✅ No complex configuration needed
- ✅ Fast (minimal logging overhead)

**Cons:**
- ⚠️ Can't disable SPAN_TRACE without potentially affecting other TRACE logs
- ⚠️ Global log level only (no per-component control)

---

## .NET (dd-trace-dotnet): Serilog Filtering Required

### Default Behavior

**With DD_TRACE_DEBUG=true:**

Logs **full span details** on start and finish:

```csharp
// Span start
[2024-02-25 10:30:45.123] [DBG] Span started: [s_id: 1234, p_id: 0, t_id: 5678] 
  with Tags: [{http.method: GET}], Tags Type: [Dictionary`2]

// Span finish
[2024-02-25 10:30:45.456] [DBG] Span closed: [s_id: 1234, p_id: 0, t_id: 5678] 
  for (Service: my-service, Resource: GET /api/users, Operation: http.request, 
  Tags: [{http.method: GET, http.url: /api/users}])
  Details:Name: http.request, Service: my-service, ...
```

**Location:** `/tracer/src/Datadog.Trace/Span.cs`

### Solution 1: Environment Variable + Code

**Custom env var parsing:**

```csharp
using Serilog;
using Serilog.Events;
using Datadog.Trace.Logging;

// Read custom env var
var spanLogLevel = Environment.GetEnvironmentVariable("DD_SPAN_LOG_LEVEL") ?? "Information";

var loggerConfiguration = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .MinimumLevel.Override("Datadog.Trace.Span", 
                          Enum.Parse<LogEventLevel>(spanLogLevel));

// Apply to Datadog logging
DatadogLogging.UseCustomLogger(loggerConfiguration.CreateLogger());
```

**Environment variables:**

```bash
export DD_TRACE_DEBUG=true
export DD_SPAN_LOG_LEVEL=Warning  # Custom: suppress span logs
```

### Solution 2: Pure Serilog Filtering (Code Only)

```csharp
using Serilog;
using Serilog.Events;

var loggerConfiguration = new LoggerConfiguration()
    .MinimumLevel.Debug()
    
    // Suppress Span logging
    .MinimumLevel.Override("Datadog.Trace.Span", LogEventLevel.Warning)
    
    // Keep other components at debug
    .MinimumLevel.Override("Datadog.Trace.Debugger", LogEventLevel.Debug)
    .MinimumLevel.Override("Datadog.Trace.AppSec", LogEventLevel.Debug)
    .MinimumLevel.Override("Datadog.Trace.Profiler", LogEventLevel.Debug);
```

### Solution 3: Serilog Filter Expression

```csharp
using Serilog;
using Serilog.Filters;

var loggerConfiguration = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .Filter.ByExcluding(logEvent =>
    {
        // Filter out span start/close messages
        var message = logEvent.MessageTemplate.Text;
        return message.Contains("Span started:") || 
               message.Contains("Span closed:");
    });
```

### Result

**Logs you WILL see:**
```
[2024-02-25 10:30:45.123] [DBG] Dynamic Instrumentation.InstrumentProbes: Request to instrument 5 probes
[2024-02-25 10:30:45.124] [DBG] AppSec: WAF initialized successfully
[2024-02-25 10:30:45.125] [DBG] ProfilerManager: Starting profiler
```

**Logs you WON'T see:**
```
[2024-02-25 10:30:45.456] [DBG] Span started: [s_id: 1234, ...]
[2024-02-25 10:30:45.789] [DBG] Span closed: [s_id: 1234, ...]
```

### Pros & Cons

**Pros:**
- ✅ Serilog provides powerful filtering
- ✅ Can use namespace overrides or message filters
- ✅ Flexible (per-type, per-message, etc.)

**Cons:**
- ❌ Requires code changes
- ❌ No built-in env var support
- ⚠️ Must redeploy application to change

---

## Go (dd-trace-go): No Component Control

### Default Behavior

**With DD_TRACE_DEBUG=true:**

Logs **full span details** with guard checks:

```go
// File: ddtrace/tracer/tracer.go:806-809
if log.DebugEnabled() {
    log.Debug("Started Span: %v, Operation: %s, Resource: %s, Tags: %v, %v",
        span, span.name, span.resource, span.meta, span.metrics)
}

// File: ddtrace/tracer/span.go:941-944
if log.DebugEnabled() {
    log.Debug("Finished Span: %v, Operation: %s, Resource: %s, Tags: %v, %v",
        span, s.name, s.resource, s.meta, s.metrics)
}
```

### Solution: Not Possible Without Code Changes

**Option 1: Fork and modify tracer**

```go
// Modify internal/log/log.go to add span filtering
var suppressSpanLogs = os.Getenv("DD_SUPPRESS_SPAN_LOGS") == "true"

func Debug(msg string, args ...interface{}) {
    if suppressSpanLogs && strings.Contains(msg, "Span:") {
        return
    }
    
    if levelThreshold > LevelDebug {
        return
    }
    logger.Log(fmt.Sprintf(msg, args...))
}
```

**Option 2: Custom logger**

```go
import (
    "gopkg.in/DataDog/dd-trace-go.v1/ddtrace/tracer"
    "strings"
)

// Custom logger that filters span messages
type FilteredLogger struct {
    underlying Logger
}

func (l *FilteredLogger) Log(msg string) {
    // Filter out span logging
    if strings.Contains(msg, "Started Span:") || strings.Contains(msg, "Finished Span:") {
        return
    }
    l.underlying.Log(msg)
}

// Use custom logger
tracer.Start(
    tracer.WithLogger(&FilteredLogger{underlying: defaultLogger}),
    tracer.WithDebugMode(true),
)
```

### Option 3: Wrapper Function (Workaround)

```go
import (
    ddlog "gopkg.in/DataDog/dd-trace-go.v1/internal/log"
    "os"
    "strings"
)

// Set up logger wrapper
type spanSuppressingLogger struct {
    original ddlog.Logger
}

func (l *spanSuppressingLogger) Log(msg string) {
    if strings.Contains(msg, "Span:") {
        return
    }
    l.original.Log(msg)
}

func init() {
    if os.Getenv("DD_SUPPRESS_SPAN_LOGS") == "true" {
        // Note: This requires access to internal package
        // Not recommended for production
    }
}
```

### Limitations

**Cannot be done without:**
- ❌ Forking the tracer
- ❌ Modifying internal packages
- ❌ Custom logger wrapper (brittle)

### Recommended: Keep Global at WARN

```bash
# Don't use DD_TRACE_DEBUG if you don't want span logs
export DD_TRACE_DEBUG=false

# Use specific feature flags
export DD_TRACE_DEBUG_ABANDONED_SPANS=true
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=true
```

**Trade-off:** Loses debug logs from all components.

### Pros & Cons

**Pros:**
- (None - not possible)

**Cons:**
- ❌ No component control
- ❌ All-or-nothing debug mode
- ❌ Would require forking tracer
- ❌ No env var solution

---

## JavaScript (dd-trace-js): Log Level Thresholds

### Default Behavior

**With DD_TRACE_DEBUG=true (level 20):**

Operational messages only:
```
DEBUG: Span processor started
DEBUG: Instrumentation loaded for http
```

**With trace level (level 10):**

Full span objects logged:
```javascript
// log/index.js:54-70
function trace(message, params) {
  // Logs complete span objects with all properties
  log(10, message, params)  // Level 10 = TRACE
}
```

### Solution: Set Debug Level, Avoid Trace Level

```bash
# Debug level (20) - no span objects
export DD_TRACE_DEBUG=true

# Ensure not at trace level
# (trace level would log full span objects)
```

### Programmatic Control

```javascript
const tracer = require('dd-trace')

tracer.init({
  logLevel: 'debug',  // 20 - no span objects
  // NOT 'trace' (10) - would log full spans
  
  debug: true
})
```

### Custom Logger with Span Filtering

```javascript
const tracer = require('dd-trace')

const customLogger = {
  trace: () => {},  // Suppress trace level (spans)
  debug: (msg) => console.log('[DEBUG]', msg),
  info: (msg) => console.log('[INFO]', msg),
  warn: (msg) => console.warn('[WARN]', msg),
  error: (msg) => console.error('[ERROR]', msg)
}

tracer.init({
  logger: customLogger,
  logLevel: 'debug'
})
```

### Diagnostic Channel Filtering

```javascript
const dc = require('diagnostics_channel')

// Subscribe to all channels EXCEPT trace
dc.subscribe('datadog:log:debug', (msg) => console.log('[DEBUG]', msg.message))
dc.subscribe('datadog:log:info', (msg) => console.log('[INFO]', msg.message))
dc.subscribe('datadog:log:warn', (msg) => console.warn('[WARN]', msg.message))
dc.subscribe('datadog:log:error', (msg) => console.error('[ERROR]', msg.message))

// DON'T subscribe to 'datadog:log:trace' - this suppresses span logging

tracer.init({
  logLevel: 'trace',  // Enable all levels
  // But channel isn't subscribed, so trace logs are suppressed
})
```

### Result

**Logs you WILL see:**
```
DEBUG: Debugger client started
DEBUG: Probe applied: probe_id=abc123
DEBUG: AppSec rule matched
```

**Logs you WON'T see:**
```
TRACE: <Span object with all properties>
```

### Pros & Cons

**Pros:**
- ✅ Simple log level control
- ✅ Span objects only at trace level
- ✅ Custom logger support
- ✅ Diagnostic channel filtering

**Cons:**
- ⚠️ Trace level logs full objects (must avoid this level)
- ⚠️ No per-component env var control

---

## Comparison Summary

### Easiest Solutions

**🥇 Python, PHP:** Already don't log full span details  
**🥈 JavaScript:** Simple log level threshold  
**🥉 Java:** Per-package system properties

### Most Difficult

**🏴 Ruby:** Custom logger filter required  
**🏴 .NET:** Serilog filtering (code changes)  
**🏴 Go:** Not possible without forking

### Recommended Approach by Tracer

| Tracer | Recommendation |
|--------|---------------|
| **Ruby** | Implement custom logger filter (example provided) |
| **Python** | No action needed - already production-safe |
| **Java** | Use `-Ddatadog.slf4j.simpleLogger.log.datadog.trace.core=warn` |
| **PHP** | No action needed - already minimal |
| **.NET** | Use Serilog namespace override (code required) |
| **Go** | Keep global at WARN; don't use DD_TRACE_DEBUG |
| **JavaScript** | Keep at debug level (20), avoid trace level (10) |

---

## Real-World Configuration Examples

### Scenario: Debug DI Only (No Spans)

**Ruby:**
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true
  c.dynamic_instrumentation.internal.trace_logging = true
  
  # Apply span filter (see Ruby section for full implementation)
  c.logger.instance = span_filtering_logger
end
```

**Python:**
```bash
export DD_TRACE_LOG_LEVEL=INFO
export _DD_DEBUGGING_LOG_LEVEL=DEBUG
```

**Java:**
```bash
java -Ddatadog.slf4j.simpleLogger.log.datadog.trace.core=warn \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=debug \
     -javaagent:dd-java-agent.jar
```

**PHP:**
```bash
export DD_TRACE_LOG_LEVEL=debug
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=1
# Span logs already minimal
```

**.NET:**
```csharp
var config = new LoggerConfiguration()
    .MinimumLevel.Override("Datadog.Trace.Span", LogEventLevel.Warning)
    .MinimumLevel.Override("Datadog.Trace.Debugger", LogEventLevel.Debug);
```

**Go:**
```bash
# Not easily possible - avoid DD_TRACE_DEBUG
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=true
```

**JavaScript:**
```javascript
tracer.init({
  logLevel: 'debug',  // Not 'trace'
  debug: true
})
```

---

### Scenario: Debug AppSec + DI (No Spans)

**Python (Best):**
```bash
export DD_TRACE_LOG_LEVEL=WARNING
export _DD_DEBUGGING_LOG_LEVEL=DEBUG
export _DD_APPSEC_LOG_LEVEL=DEBUG
```

**Java (Good):**
```bash
java -Ddatadog.slf4j.simpleLogger.defaultLogLevel=warn \
     -Ddatadog.slf4j.simpleLogger.log.datadog.trace.core=warn \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=debug \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.appsec=debug
```

**.NET (Requires Code):**
```csharp
var config = new LoggerConfiguration()
    .MinimumLevel.Warning()
    .MinimumLevel.Override("Datadog.Trace.Span", LogEventLevel.Warning)
    .MinimumLevel.Override("Datadog.Trace.Debugger", LogEventLevel.Debug)
    .MinimumLevel.Override("Datadog.Trace.AppSec", LogEventLevel.Debug);
```

---

## Performance Impact

### With Span Logging Enabled

| Tracer | Overhead per span | Impact on high-throughput |
|--------|------------------|--------------------------|
| **Ruby** | High (50-150 lines) | 🔴 Significant (100+ MB/s logs) |
| **Go** | High (15-40 lines) | 🟡 Moderate (50+ MB/s logs) |
| **.NET** | Moderate (10-30 lines) | 🟡 Moderate (30+ MB/s logs) |
| **Java** | Low (5-10 lines) | 🟢 Low (10+ MB/s logs) |
| **JavaScript** | Variable (50+ with trace) | 🟡 Moderate at trace level |
| **Python** | Very Low (2-5 lines) | 🟢 Minimal (5 MB/s logs) |
| **PHP** | Very Low (3-8 lines) | 🟢 Minimal (5 MB/s logs) |

**Impact:** At 1000 requests/second with 3 spans each:
- **Ruby:** ~150,000 log lines/sec = ~750 MB/s
- **Go:** ~120,000 log lines/sec = ~480 MB/s
- **.NET:** ~90,000 log lines/sec = ~360 MB/s
- **Python/PHP:** ~15,000 log lines/sec = ~60 MB/s

### With Span Logging Disabled

All tracers: < 100 log lines/sec for non-span debug logs = ~1-5 MB/s

**Reduction:** 99% reduction in log volume for high-verbosity tracers

---

## Recommendations

### Production Environments

1. **Always disable span logging in production** unless actively debugging trace data issues
2. **Use component-specific logging** (Python, Java) to debug individual features
3. **Monitor log volume** - set up alerts for excessive logging
4. **Use APM UI** instead of logs for span inspection

### Development Environments

1. **Ruby, Go, .NET:** Consider disabling span logging even in dev for high-throughput testing
2. **Python, PHP:** Safe to use debug mode (spans not verbose)
3. **Java:** Use TraceStructureWriter only when specifically debugging trace structure

### Debugging Workflow

**Step 1:** Debug with span logging disabled
```bash
# Python example
export _DD_DEBUGGING_LOG_LEVEL=DEBUG
export _DD_APPSEC_LOG_LEVEL=DEBUG
export DD_TRACE_LOG_LEVEL=WARNING
```

**Step 2:** If trace data issue suspected, enable span logging temporarily
```bash
# Ruby example (requires custom filter removal)
# Go example (enable DD_TRACE_DEBUG briefly)
```

**Step 3:** Disable span logging again after debugging

---

## Future Improvements

### What Tracers Could Add

**Ruby:**
- Built-in env var: `DD_TRACE_LOG_SPANS=false`
- Built-in component control like Python

**Go:**
- Per-component log levels
- `DD_TRACE_LOG_SPANS=false` env var

**.NET:**
- Built-in env var: `DD_SPAN_LOG_LEVEL=Warning`
- Pre-configured Serilog namespace overrides

**PHP:**
- Per-component log level control
- Env var to disable SPAN_TRACE specifically

---

## Conclusion

**Best tracers for this use case:**
1. **Python** - No span details logged by default, excellent component control
2. **Java** - Easy per-package control, minimal span logging
3. **PHP** - Already minimal span logging, no action needed

**Most challenging:**
1. **Go** - No solution without forking
2. **Ruby** - Requires complex custom logger filter
3. **.NET** - Requires code changes for Serilog filtering

**Key takeaway:** If you need to debug non-span components in production while keeping logs manageable, **Python and Java** are the clear winners with their environment variable-based component control.
