# Datadog Tracer Logging: Complete Comparison (All 7 Tracers)

## Overview

This document compares logging configuration and behavior across all 7 Datadog tracers:
- **dd-trace-rb** (Ruby)
- **dd-trace-py** (Python)
- **dd-trace-java** (Java)
- **dd-trace-php** (PHP)
- **dd-trace-dotnet** (.NET/C#)
- **dd-trace-go** (Go)
- **dd-trace-js** (JavaScript/Node.js)

---

## Major Differences Summary

| Feature | Ruby | Python | Java | PHP | .NET | Go | JavaScript |
|---------|------|--------|------|-----|------|----|-----------| 
| **Span detail logging** | ✅ Full pretty_inspect | ❌ Operational only | ⚠️ Optional | ❌ IDs only | ✅ Full details | ✅ Full details | ⚠️ Trace level only |
| **DD_TRACE_DEBUG behavior** | DEBUG + verbose spans | DEBUG only | DEBUG + dynamic | DEBUG + JSON | DEBUG + AppDomain | DEBUG + JSON | DEBUG + diagnostics |
| **Component control** | Programmatic only | Env var trie | System properties | Component bitfield | Per-component loggers | Abandoned spans flag | Diagnostic channels |
| **Hierarchical filtering** | ❌ Not built-in | ✅ Prefix trie | ✅ Package hierarchy | ✅ Component bitfield | ✅ Per-type loggers | ❌ Global level | ✅ Diagnostic channels |
| **Rate limiting** | ❌ No | ✅ 60s per-log | ⚠️ DI only | ✅ Trace-level 100/s | ✅ Per-message | ✅ Error aggregation 60s | ⚠️ Deprecations only |
| **DI verbose logging** | `trace_logging` | `_DD_DEBUGGING_LOG_LEVEL` | Package properties | Same logger | Per-probe logging | Remote Config logs | Debugger client logs |
| **Configuration approach** | Ruby code | Env vars | System properties | Env vars | Env vars | Env vars | Env vars + Fleet |
| **Logger framework** | Custom | Python logging | Custom SLF4J | C ext + Rust | Serilog | Custom | Custom (pluggable) |
| **JSON output** | ❌ No | ❌ No | ✅ Yes | ⚠️ Userland only | ✅ Startup only | ✅ Startup only | ❌ No |
| **Dynamic switching** | ❌ No | ❌ No | ✅ Remote Config | ⚠️ INI runtime | ❌ No | ❌ No | ⚠️ Fleet Config |
| **Log rotation** | ❌ No | ❌ No | ❌ No | ✅ 60s atomic | ✅ Size-based | ❌ No | ❌ No |
| **Startup config dump** | ❌ No | ❌ No | ❌ No | ✅ JSON | ✅ JSON | ✅ JSON | ✅ Summary |
| **Telemetry integration** | ❌ No | ❌ No | ❌ No | ❌ No | ✅ Yes | ✅ Yes | ❌ No |

---

## Verbosity Comparison

| Tracer | Lines per trace (debug mode) | Includes full span data? |
|--------|------------------------------|-------------------------|
| **Ruby** | ~100-150 | ✅ Yes (pretty_inspect) |
| **Python** | ~10-20 | ❌ No (operational only) |
| **Java** | ~5-10 (50-100 with writer) | ⚠️ Optional (TraceStructureWriter) |
| **PHP** | ~3-8 | ❌ No (trace_id/span_id only) |
| **.NET** | ~10-30 | ✅ Yes (full span object logged) |
| **Go** | ~15-40 | ✅ Yes (name, resource, tags, metrics) |
| **JavaScript** | ~5-15 (50+ with trace level) | ⚠️ Trace level only |

**Most verbose:** Ruby (~100-150 lines/trace)  
**Least verbose:** PHP (~3-8 lines/trace)  
**Balanced:** Python, Java, JavaScript (~5-20 lines/trace)

---

## DD_TRACE_DEBUG Behavior

### Ruby
```bash
export DD_TRACE_DEBUG=true
```
- Sets logger to DEBUG
- **Enables span pretty_inspect** (~50-150 lines per trace)
- Adds caller location to all logs
- Enables DI trace_logging

### Python
```bash
export DD_TRACE_DEBUG=1
```
- Sets logger to DEBUG
- **Does NOT log span details** (by design)
- Disables rate limiting
- Operational messages only

### Java
```bash
export DD_TRACE_DEBUG=true
```
- Sets logger to DEBUG
- **Does NOT log spans by default**
- Can enable TraceStructureWriter
- Supports JSON format
- Dynamic switching via Remote Config

### PHP
```bash
export DD_TRACE_DEBUG=1
```
- Sets logger to DEBUG/TRACE
- **Startup JSON config dump** (once)
- Logs span lifecycle (trace_id, span_id)
- Automatic 60s log rotation
- No full span details

### .NET
```bash
export DD_TRACE_DEBUG=true
```
- Sets Serilog to DEBUG
- **Logs full span details** on start/close
- AppDomain enrichment
- JSON startup diagnostic
- Telemetry integration
- Size-based log rotation (10MB)

### Go
```bash
export DD_TRACE_DEBUG=true
```
- Sets log level to DEBUG
- **Logs complete span details** (name, resource, all tags/metrics)
- JSON startup configuration
- Error aggregation (60s default)
- Abandoned spans detection

### JavaScript
```bash
export DD_TRACE_DEBUG=true
```
- Sets log level to DEBUG (20)
- **Operational messages** at debug level
- **Full span objects** only at TRACE level (10)
- Diagnostic channels enabled
- Startup configuration summary
- Custom logger support (Winston, Bunyan)

---

## Component-Specific Logging Control

### Ruby
**Method:** Programmatic only
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true
  c.dynamic_instrumentation.internal.trace_logging = false
  # Requires custom logger filter for component isolation
end
```

### Python
**Method:** Hierarchical env vars
```bash
export _DD_DEBUGGING_LOG_LEVEL=DEBUG          # DI only
export _DD_TRACE_PROCESSOR_LOG_LEVEL=DEBUG    # Trace processor
export _DD_APPSEC_LOG_LEVEL=DEBUG             # AppSec
```

### Java
**Method:** Package hierarchy system properties
```bash
java -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=debug \
     -Ddatadog.slf4j.simpleLogger.log.datadog.trace.core=debug
```

### PHP
**Method:** Global level + feature flags
```bash
export DD_TRACE_LOG_LEVEL=debug               # Global only
export DD_TRACE_STARTUP_LOGS=0                # Disable startup
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=0   # Disable DI
```

### .NET
**Method:** Per-type logger retrieval
```csharp
// Each type gets its own logger instance
var log = DatadogLogging.GetLoggerFor<MyClass>();
log.Debug("Component-specific message");
```
**Note:** No per-component env var control; uses Serilog filtering

### Go
**Method:** Feature-specific flags
```bash
export DD_TRACE_DEBUG=true                        # Global debug
export DD_TRACE_DEBUG_ABANDONED_SPANS=true        # Abandoned span logging
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=true    # DI remote config
```
**Note:** No per-component level control; all at global level

### JavaScript
**Method:** Diagnostic channels + custom loggers
```javascript
// Subscribe to specific log channels
dc.subscribe('datadog:log:debug', (msg) => { /* ... */ });
dc.subscribe('datadog:log:error', (msg) => { /* ... */ });

// Or use custom logger
tracer.use('my-plugin', {
  logInjection: true,
  logger: myCustomLogger  // Winston, Bunyan, etc.
});
```

---

## Rate Limiting

| Tracer | Mechanism | Default | Configurable |
|--------|-----------|---------|--------------|
| **Ruby** | ❌ None | N/A | ❌ No |
| **Python** | ✅ Per-location | 60s | ✅ `DD_TRACE_LOGGING_RATE` |
| **Java** | ⚠️ DI probes only | 1/s snapshots, 5000/s logs | ✅ System properties |
| **PHP** | ✅ Trace-level | 100 traces/s | ✅ `DD_TRACE_RATE_LIMIT` |
| **.NET** | ✅ Per-message | Configurable | ✅ `DD_TRACE_LOGGING_RATE` |
| **Go** | ✅ Error aggregation | 60s | ✅ `DD_LOGGING_RATE` |
| **JavaScript** | ⚠️ Deprecations only | Once per code | ❌ No (memoized) |

**Best rate limiting:** Python (per-location, 60s), .NET (per-message with skip count)  
**Least rate limiting:** Ruby (none), JavaScript (deprecations only)

---

## JSON Logging Support

| Tracer | Support Level | Format | Configuration |
|--------|--------------|--------|---------------|
| **Ruby** | ❌ None | N/A | N/A |
| **Python** | ❌ None | N/A | N/A |
| **Java** | ✅ Full | Structured JSON | `dd.log.format.json=true` |
| **PHP** | ⚠️ Userland only | DatadogLogger JSON | PHP API only |
| **.NET** | ⚠️ Startup only | Diagnostic JSON | Serilog template |
| **Go** | ⚠️ Startup only | Configuration JSON | Automatic with `DD_TRACE_STARTUP_LOGS` |
| **JavaScript** | ❌ None | Util.format | Compatible with Winston/Bunyan |

**Best JSON support:** Java (full structured logging)  
**Limited support:** PHP (userland), .NET/Go (startup only)  
**No support:** Ruby, Python, JavaScript

---

## Dynamic Log Level Switching

| Tracer | Capability | Mechanism | Requires Restart |
|--------|-----------|-----------|------------------|
| **Ruby** | ❌ No | N/A | ✅ Yes |
| **Python** | ❌ No | N/A | ✅ Yes |
| **Java** | ✅ Yes | Remote Config | ❌ No |
| **PHP** | ⚠️ Partial | INI runtime | ⚠️ CLI: Yes, Web: No |
| **.NET** | ❌ No | N/A | ✅ Yes |
| **Go** | ❌ No | N/A | ✅ Yes |
| **JavaScript** | ⚠️ Partial | Fleet Config | ⚠️ Process-level only |

**Best dynamic switching:** Java (full Remote Config integration)  
**Partial support:** PHP (non-CLI SAPIs), JavaScript (Fleet Config)  
**No support:** Ruby, Python, .NET, Go

---

## Log File Rotation

| Tracer | Automatic Rotation | Mechanism | Configuration |
|--------|-------------------|-----------|---------------|
| **Ruby** | ❌ No | Manual | N/A |
| **Python** | ❌ No | Manual | N/A |
| **Java** | ❌ No | Manual | N/A |
| **PHP** | ✅ Yes | 60s atomic FD | Automatic |
| **.NET** | ✅ Yes | Size-based (10MB) | `DD_MAX_LOGFILE_SIZE` |
| **Go** | ❌ No | Manual | N/A |
| **JavaScript** | ❌ No | Manual | N/A |

**Best rotation:** PHP (automatic 60s), .NET (size-based 10MB)  
**No rotation:** Ruby, Python, Java, Go, JavaScript

---

## Startup Configuration Logging

| Tracer | Startup Logs | Format | Configuration |
|--------|-------------|--------|---------------|
| **Ruby** | ❌ No | N/A | N/A |
| **Python** | ❌ No | N/A | N/A |
| **Java** | ❌ No | N/A | N/A |
| **PHP** | ✅ Yes | JSON (~500-1000 chars) | `DD_TRACE_STARTUP_LOGS` |
| **.NET** | ✅ Yes | JSON | `DD_TRACE_STARTUP_LOGS` |
| **Go** | ✅ Yes | JSON | `DD_TRACE_STARTUP_LOGS` |
| **JavaScript** | ✅ Yes | Text summary | Automatic |

**Best startup logging:** PHP, .NET, Go (comprehensive JSON)  
**Text summary:** JavaScript  
**No startup logging:** Ruby, Python, Java

---

## Configuration Scenarios

### Scenario 1: Only Tracer Logs (No DI)

```bash
# Ruby
# Requires programmatic config

# Python
export DD_TRACE_DEBUG=1
export _DD_DEBUGGING_LOG_LEVEL=WARNING

# Java
java -Ddd.trace.debug=true \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=warn

# PHP
export DD_TRACE_DEBUG=1
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=0

# .NET
export DD_TRACE_DEBUG=true
# DI logs through same logger, use filtering

# Go
export DD_TRACE_DEBUG=true
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=false

# JavaScript
export DD_TRACE_DEBUG=true
# DI logs through same system
```

### Scenario 2: Production (Minimal Noise)

```bash
# Ruby
# Avoid - too verbose

# Python
export DD_TRACE_LOG_LEVEL=WARNING

# Java
java -Ddatadog.slf4j.simpleLogger.defaultLogLevel=warn

# PHP
export DD_TRACE_LOG_LEVEL=error
export DD_TRACE_STARTUP_LOGS=1

# .NET
export DD_TRACE_LOG_LEVEL=Warning

# Go
# Default: WARN (no env var needed)

# JavaScript
# Default: WARN (no env var needed)
```

---

## Environment Variables Summary

| Variable | Ruby | Python | Java | PHP | .NET | Go | JS |
|----------|------|--------|------|-----|------|----|----|
| `DD_TRACE_DEBUG` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `DD_TRACE_LOG_LEVEL` | ❌ | ✅ | ⚠️ | ✅ | ✅ | ❌ | ⚠️ |
| `DD_TRACE_LOG_FILE` | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| `DD_TRACE_LOG_DIRECTORY` | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ |
| `DD_TRACE_STARTUP_LOGS` | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ |
| `DD_TRACE_LOGGING_RATE` | ❌ | ✅ | ❌ | ❌ | ✅ | ⚠️ `DD_LOGGING_RATE` | ❌ |
| `DD_TRACE_RATE_LIMIT` | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| `_DD_<COMPONENT>_LOG_LEVEL` | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

---

## Best Practices by Tracer

### Ruby
**Development:**
```ruby
Datadog.configure { |c| c.diagnostics.debug = true }
```
**Production:** ❌ Avoid - too verbose

### Python
**Development:**
```bash
export DD_TRACE_DEBUG=1
export DD_TRACE_LOGGING_RATE=0
```
**Production:**
```bash
export DD_TRACE_LOG_LEVEL=WARNING
export _DD_DEBUGGING_LOG_LEVEL=DEBUG  # If needed
```

### Java
**Development:**
```bash
java -Ddd.trace.debug=true
```
**Production:**
```bash
java -Ddatadog.slf4j.simpleLogger.defaultLogLevel=warn \
     -Ddd.log.format.json=true
```

### PHP
**Development:**
```bash
export DD_TRACE_DEBUG=1
export DD_TRACE_ONCE_LOGS=0
```
**Production:**
```bash
export DD_TRACE_LOG_LEVEL=error
export DD_TRACE_STARTUP_LOGS=1
export DD_TRACE_LOG_FILE=/var/log/datadog-php.log
```

### .NET
**Development:**
```bash
export DD_TRACE_DEBUG=true
```
**Production:**
```bash
export DD_TRACE_LOG_LEVEL=Warning
# Automatic log rotation at 10MB
```

### Go
**Development:**
```bash
export DD_TRACE_DEBUG=true
export DD_LOGGING_RATE=0  # Disable error aggregation
```
**Production:**
```bash
# Default WARN level
export DD_LOGGING_RATE=60  # Error aggregation every 60s
```

### JavaScript
**Development:**
```bash
export DD_TRACE_DEBUG=true
# Or programmatically:
# tracer.init({ debug: true, logLevel: 'debug' })
```
**Production:**
```bash
# Default WARN level
# Or set programmatically:
# tracer.init({ logLevel: 'error' })
```

---

## Language-Specific Implementation

### Ruby
- **Language:** Pure Ruby
- **Logger:** `Datadog::Core::Logger`
- **Span logging:** `/dtr/lib/datadog/tracing/tracer.rb:566-567`
- **Verbosity:** Highest (~100-150 lines/trace)

### Python
- **Language:** Pure Python
- **Logger:** Python `logging` module
- **Hierarchical:** Prefix trie matching
- **Verbosity:** Low (~10-20 lines/trace)

### Java
- **Language:** Pure Java
- **Logger:** Custom SLF4J
- **Package hierarchy:** Walks up package tree
- **Verbosity:** Low (~5-10 lines/trace, 50-100 with writer)

### PHP
- **Language:** C extension + Rust FFI + PHP
- **Logger:** Custom C logging + DatadogLogger (userland)
- **Rotation:** Automatic 60s atomic
- **Verbosity:** Lowest (~3-8 lines/trace)

### .NET
- **Language:** C# / .NET
- **Logger:** Serilog (vendored)
- **Enrichment:** AppDomain, Process, MachineName
- **Verbosity:** Medium (~10-30 lines/trace)

### Go
- **Language:** Pure Go
- **Logger:** Custom with error aggregation
- **Optimization:** Debug guards to avoid allocation
- **Verbosity:** Medium (~15-40 lines/trace)

### JavaScript
- **Language:** Pure JavaScript/Node.js
- **Logger:** Custom with pluggable interface
- **Channels:** Diagnostic channels for subscriptions
- **Verbosity:** Low-Medium (~5-15 lines/trace, 50+ with trace level)

---

## Strengths & Weaknesses Summary

### Ruby
**✅ Strengths:** Complete span introspection, full trace data visibility  
**❌ Weaknesses:** Very verbose, no component filtering, production risk

### Python
**✅ Strengths:** Granular control, rate limiting, performance-focused  
**❌ Weaknesses:** No span details, operational only

### Java
**✅ Strengths:** Dynamic switching, JSON output, SLF4J compatible, balanced  
**❌ Weaknesses:** Complex config, requires SLF4J knowledge

### PHP
**✅ Strengths:** Lightweight, automatic rotation, startup JSON, runtime INI  
**❌ Weaknesses:** No component filtering, global level only, no span details

### .NET
**✅ Strengths:** Serilog integration, telemetry, size-based rotation, structured  
**❌ Weaknesses:** No remote config, complex enrichment, large log files

### Go
**✅ Strengths:** Error aggregation, JSON startup, performance guards, simple  
**❌ Weaknesses:** No component filtering, no remote config, manual rotation

### JavaScript
**✅ Strengths:** Pluggable loggers, diagnostic channels, custom logger support  
**❌ Weaknesses:** No JSON native, limited rate limiting, no rotation

---

## Quick Reference Matrix

| Use Case | Best Tracer | Reason |
|----------|-------------|--------|
| **Full span inspection** | Ruby, .NET, Go | Complete span details logged |
| **Production safety** | Python, PHP, Java | Minimal noise, rate limiting |
| **JSON logging** | Java | Full structured JSON support |
| **Log rotation** | PHP, .NET | Automatic rotation built-in |
| **Component filtering** | Python, Java | Hierarchical control |
| **Dynamic debugging** | Java | Remote Config integration |
| **Lightweight logging** | PHP | Most minimal output |
| **Startup auditing** | PHP, .NET, Go | Comprehensive JSON config |
| **Custom logger integration** | JavaScript | Pluggable logger interface |
| **Rate limiting** | Python, .NET | Per-location/per-message limiting |

---

## Migration Recommendations

### From Ruby
- **To Python:** Expect less verbose output, use APM UI for span details
- **To Java:** Similar verbosity with optional writer, JSON support
- **To .NET:** Similar verbosity, better structured logging
- **To Go:** Similar verbosity, simpler configuration
- **To JavaScript:** Similar verbosity, more flexible logger integration

### From Python
- **To Java:** Similar operational logging, add JSON support
- **To .NET:** Gain startup logs, lose hierarchical env vars
- **To Go:** Gain startup logs, lose per-component control
- **To JavaScript:** Similar operational approach, custom logger options

### From Java
- **To .NET:** Lose dynamic switching, gain Serilog integration
- **To Go:** Lose component control, gain simpler config
- **To JavaScript:** Lose JSON, gain pluggable loggers

### From PHP
- **To .NET:** Gain component control, lose automatic rotation
- **To Go:** Similar startup logs, gain error aggregation
- **To JavaScript:** Lose rotation, gain flexible loggers

---

## Code Locations Reference

### Ruby
- Span logging: `/dtr/lib/datadog/tracing/tracer.rb:566-567`
- Logger: `/dtr/lib/datadog/core/logger.rb`

### Python
- Main config: `/dd-trace-py/ddtrace/_logger.py`
- Hierarchical: `/dd-trace-py/ddtrace/internal/logger.py`

### Java
- Config: `/dd-trace-java/internal-api/src/main/java/datadog/trace/api/Config.java`
- Logger: `/dd-trace-java/dd-java-agent/agent-logging/`

### PHP
- C Logging: `/dd-trace-php/ext/logging.c`, `/dd-trace-php/ext/logging.h`
- Config: `/dd-trace-php/ext/configuration.h`
- Span: `/dd-trace-php/ext/span.c`

### .NET
- Serilog: `/dd-trace-dotnet/tracer/src/Datadog.Trace/Logging/Internal/DatadogSerilogLogger.cs`
- Startup: `/dd-trace-dotnet/tracer/src/Datadog.Tracer.Native/StartupLogger.cs`
- Span: `/dd-trace-dotnet/tracer/src/Datadog.Trace/Span.cs`

### Go
- Main log: `/dd-trace-go/internal/log/log.go`
- Config: `/dd-trace-go/internal/config/config.go`
- Span: `/dd-trace-go/ddtrace/tracer/span.go`, `/dd-trace-go/ddtrace/tracer/tracer.go`

### JavaScript
- Main log: `/dd-trace-js/packages/dd-trace/src/log/index.js`
- Channels: `/dd-trace-js/packages/dd-trace/src/log/channels.js`
- Config: `/dd-trace-js/packages/dd-trace/src/config/index.js`

---

## Summary

**Most verbose:** Ruby (~100-150 lines/trace)  
**Least verbose:** PHP (~3-8 lines/trace)  
**Best for production:** Python, PHP, Java (minimal noise with rate limiting)  
**Best for debugging:** Ruby, .NET, Go (full span details)  
**Most flexible:** Java (dynamic switching), JavaScript (pluggable loggers)  
**Best structured logging:** Java (full JSON), .NET/Go/PHP (startup JSON)  
**Best component control:** Python (hierarchical env vars), Java (package properties)

**Total documentation coverage:** 7 tracers across Ruby, Python, Java, PHP, .NET, Go, JavaScript
