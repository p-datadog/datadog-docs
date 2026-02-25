# Component-Specific Logging Control: Deep Dive Across All Tracers

## Overview

This document provides an in-depth analysis of **component-specific logging control** across all 7 Datadog tracers. Component control allows you to enable debug logging for specific parts of the tracer (e.g., only the debugger, only AppSec, only span processing) while keeping other components at a higher log level.

---

## What is Component Control?

**Component control** is the ability to configure different log levels for different subsystems of the tracer independently, rather than applying a single global log level to the entire tracer.

### Why Component Control Matters

**Without component control:**
```bash
# This enables DEBUG for EVERYTHING
export DD_TRACE_DEBUG=true
```
**Result:** Massive log output from all components: tracing, profiling, AppSec, DI, telemetry, etc.

**With component control:**
```bash
# Enable debug ONLY for Dynamic Instrumentation
export _DD_DEBUGGING_LOG_LEVEL=DEBUG  # Python
export DD_TRACE_LOG_LEVEL=INFO        # Keep everything else at INFO
```
**Result:** Only DI logs at debug level; everything else stays quiet.

### Benefits

1. **Targeted debugging** - Debug one component without noise from others
2. **Production safety** - Enable debug for specific features in production
3. **Performance** - Reduce logging overhead by limiting verbose output
4. **Log volume** - Keep log files manageable
5. **Signal-to-noise** - Find relevant logs faster

---

## Component Control by Tracer

### Summary Matrix

| Tracer | Component Control | Mechanism | Granularity | Ease of Use |
|--------|------------------|-----------|-------------|-------------|
| **Ruby** | ❌ None | Programmatic filtering | Custom | ⭐ Difficult |
| **Python** | ✅ Excellent | Env var prefix trie | Per-module | ⭐⭐⭐⭐⭐ Easy |
| **Java** | ✅ Excellent | System property hierarchy | Per-package | ⭐⭐⭐ Moderate |
| **PHP** | ❌ Limited | Global + feature flags | Feature-level | ⭐⭐ Easy |
| **.NET** | ⚠️ Partial | Per-type loggers | Per-type | ⭐⭐ Moderate |
| **Go** | ❌ None | Feature flags only | Feature-level | ⭐⭐ Easy |
| **JavaScript** | ⚠️ Partial | Diagnostic channels | Per-channel | ⭐⭐⭐ Moderate |

---

## Ruby (dd-trace-rb): Programmatic Filtering Only

### Capability: ❌ No Built-in Component Control

Ruby does **not** provide built-in environment variable-based component control. All components share the same logger level.

### Architecture

**File:** `/dtr/lib/datadog/core/logger.rb`

```ruby
module Datadog
  module Core
    class Logger
      # Single shared logger instance for all components
      def initialize(target = $stderr)
        @logger = ::Logger.new(target)
      end
    end
  end
end
```

All components use the same logger:
- Tracing
- Profiling
- AppSec
- Dynamic Instrumentation
- Remote Config
- Data Streams

### Workaround: Custom Logger Filter

**Implementation:**

```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true
  
  original_logger = c.logger.instance || Datadog::Core::Logger.new($stderr)
  
  # Create custom logger filter
  filtered_logger = Object.new.tap do |logger|
    define_singleton_method(:add) do |severity, message = nil, progname = nil, &block|
      msg = message || (block && block.call) || progname
      
      # Filter out DI logs
      return if msg.to_s.include?('di:')
      
      # Filter out profiling logs
      return if msg.to_s.include?('profiling:')
      
      # Pass through to original logger
      original_logger.add(severity, message, progname, &block)
    end
    
    # Delegate level checks
    define_singleton_method(:debug?) { original_logger.debug? }
    define_singleton_method(:info?) { original_logger.info? }
    define_singleton_method(:warn?) { original_logger.warn? }
    define_singleton_method(:error?) { original_logger.error? }
  end
  
  c.logger.instance = filtered_logger
end
```

### Example Use Cases

**Scenario 1: Only tracer logs, no DI**

```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true
  c.dynamic_instrumentation.internal.trace_logging = false
end
```

**Scenario 2: Only specific component via filter**

```ruby
# Filter: Only show AppSec logs
filtered_logger = Object.new.tap do |logger|
  define_singleton_method(:add) do |severity, message = nil, progname = nil, &block|
    msg = message || (block && block.call) || progname
    
    # ONLY log messages containing 'appsec'
    return unless msg.to_s.downcase.include?('appsec')
    
    original_logger.add(severity, message, progname, &block)
  end
end
```

### Limitations

- ❌ Requires code changes (not environment variable-based)
- ❌ String matching is brittle (depends on log message format)
- ❌ No standard component naming convention
- ❌ Must understand log message patterns
- ❌ Difficult to maintain across versions

### Strengths

- ✅ Extremely flexible (full Ruby power)
- ✅ Can implement complex filtering logic
- ✅ Can aggregate/transform logs

---

## Python (dd-trace-py): Hierarchical Env Var Control

### Capability: ✅ Excellent - Prefix Trie Matching

Python provides the **best component control** via hierarchical environment variables with prefix trie matching.

### Architecture

**File:** `/dd-trace-py/ddtrace/internal/logger.py` (lines 75-114)

```python
class LoggerPrefix:
    """Trie-based prefix matching for logger names"""
    
    def __init__(self, logger_name: str, level: str):
        self.prefix = logger_name
        self.level = getattr(logging, level.upper())
    
    @staticmethod
    def build_trie() -> "LoggerPrefixTrie":
        """Build trie from _DD_*_LOG_LEVEL env vars"""
        trie = LoggerPrefixTrie()
        
        for key, value in os.environ.items():
            if key.startswith("_DD_") and key.endswith("_LOG_LEVEL"):
                # Extract component: _DD_DEBUGGING_LOG_LEVEL -> DEBUGGING
                component = key[4:-10]  # Remove _DD_ prefix and _LOG_LEVEL suffix
                
                # Convert to logger path: DEBUGGING -> ddtrace.debugging
                logger_path = "ddtrace." + component.lower().replace("_", ".")
                
                trie.insert(logger_path, value)
        
        return trie

LOG_LEVEL_TRIE = LoggerPrefix.build_trie()

def get_logger(name: str) -> logging.Logger:
    logger = logging.getLogger(name)
    logger.addFilter(log_filter)
    
    if name.startswith("ddtrace."):
        if (level := LOG_LEVEL_TRIE.lookup(name)) is not None:
            logger.setLevel(level)
    
    return logger
```

### How It Works

1. **Environment variable format:** `_DD_<COMPONENT>_LOG_LEVEL=<LEVEL>`
2. **Component name conversion:** Underscores → dots
   - `DEBUGGING` → `ddtrace.debugging`
   - `TRACE_PROCESSOR` → `ddtrace.trace.processor`
3. **Hierarchical matching:** Applies to all submodules
   - `ddtrace.debugging` matches:
     - `ddtrace.debugging`
     - `ddtrace.debugging.probe`
     - `ddtrace.debugging.signal`
     - `ddtrace.debugging.probe.remoteconfig`

### Component Hierarchy

**Tracer Components:**
```
ddtrace._trace
├── ddtrace._trace.tracer
├── ddtrace._trace.span
├── ddtrace._trace.processor
└── ddtrace._trace.context
```

**Dynamic Instrumentation:**
```
ddtrace.debugging
├── ddtrace.debugging._debugger
├── ddtrace.debugging._signal
│   └── ddtrace.debugging._signal.tracing
└── ddtrace.debugging._probe
    └── ddtrace.debugging._probe.registry
```

**AppSec:**
```
ddtrace.appsec
├── ddtrace.appsec._iast
├── ddtrace.appsec._asm
└── ddtrace.appsec.processor
```

**Profiling:**
```
ddtrace.profiling
├── ddtrace.profiling.collector
├── ddtrace.profiling.scheduler
└── ddtrace.profiling.exporter
```

### Example Use Cases

**Scenario 1: Debug only DI**

```bash
export DD_TRACE_LOG_LEVEL=INFO
export _DD_DEBUGGING_LOG_LEVEL=DEBUG

python app.py
```

**Result:**
- `ddtrace.debugging.*` → DEBUG
- Everything else → INFO

---

**Scenario 2: Debug tracer + DI, silence AppSec**

```bash
export DD_TRACE_DEBUG=1
export _DD_APPSEC_LOG_LEVEL=ERROR

python app.py
```

**Result:**
- All components → DEBUG (from `DD_TRACE_DEBUG`)
- `ddtrace.appsec.*` → ERROR (overrides global)

---

**Scenario 3: Debug only span processor**

```bash
export DD_TRACE_LOG_LEVEL=WARNING
export _DD_TRACE_PROCESSOR_LOG_LEVEL=DEBUG

python app.py
```

**Result:**
- `ddtrace._trace.processor` → DEBUG
- Everything else → WARNING

---

**Scenario 4: Multiple component debugging**

```bash
export DD_TRACE_LOG_LEVEL=INFO
export _DD_DEBUGGING_LOG_LEVEL=DEBUG
export _DD_PROFILING_LOG_LEVEL=DEBUG
export _DD_APPSEC_IAST_LOG_LEVEL=DEBUG

python app.py
```

**Result:**
- `ddtrace.debugging.*` → DEBUG
- `ddtrace.profiling.*` → DEBUG
- `ddtrace.appsec._iast.*` → DEBUG
- Everything else → INFO

---

**Scenario 5: Fine-grained sub-component**

```bash
export DD_TRACE_LOG_LEVEL=WARNING
export _DD_DEBUGGING_SIGNAL_TRACING_LOG_LEVEL=DEBUG

python app.py
```

**Result:**
- `ddtrace.debugging._signal.tracing` → DEBUG
- `ddtrace.debugging._signal` (other) → WARNING
- Everything else → WARNING

### Discovering Component Names

**Method 1: Read the code**

Look at import paths:
```python
# File: ddtrace/debugging/_debugger.py
from ddtrace.internal.logger import get_logger
log = get_logger(__name__)  # __name__ = "ddtrace.debugging._debugger"
```

**Method 2: Enable debug and observe**

```bash
export DD_TRACE_DEBUG=1
python app.py 2>&1 | grep "^\[" | cut -d' ' -f2 | sort -u
```

Look for logger names in output like:
```
[ddtrace._trace.tracer]
[ddtrace.debugging._debugger]
[ddtrace.appsec._iast.processor]
```

**Method 3: Search codebase**

```bash
cd dd-trace-py
grep -r "get_logger(__name__)" --include="*.py" | cut -d: -f1 | xargs dirname | sort -u
```

### Strengths

- ✅ **Best-in-class component control**
- ✅ Environment variable-based (no code changes)
- ✅ Hierarchical (applies to submodules)
- ✅ Discoverable (logger names = module paths)
- ✅ Production-safe (no restart required for debugging)
- ✅ Clean syntax (`_DD_<COMPONENT>_LOG_LEVEL`)

### Limitations

- ⚠️ Must know component paths
- ⚠️ Underscore-to-dot conversion not always intuitive
- ⚠️ No wildcard support (must specify full prefix)

---

## Java (dd-trace-java): Package Hierarchy System Properties

### Capability: ✅ Excellent - Package-Based Control

Java provides component control via SLF4J system properties with package hierarchy matching.

### Architecture

**File:** `/dd-trace-java/dd-java-agent/agent-logging/src/main/java/datadog/trace/logging/simplelogger/SimpleLogger.java`

```java
private static LogLevel recursivelyComputeLogLevel(String name) {
    String tempName = name;
    
    // Walk up package hierarchy
    while (tempName != null) {
        // Check system property: datadog.slf4j.simpleLogger.log.<package>
        String levelStr = getStringProperty(
            SimpleLoggerConfiguration.LOG_KEY_PREFIX + tempName, 
            null
        );
        
        if (levelStr != null) {
            return LogLevel.valueOf(levelStr.toUpperCase());
        }
        
        // Move up one level in package hierarchy
        int lastDot = tempName.lastIndexOf(".");
        if (lastDot == -1) {
            break;
        }
        tempName = tempName.substring(0, lastDot);
    }
    
    // Fall back to default log level
    return defaultLogLevel;
}
```

**Configuration prefix:** `datadog.slf4j.simpleLogger.log.<package.name>=<level>`

### How It Works

1. **Check full package path first**
2. **Walk up hierarchy** until a match is found
3. **Fall back to default** if no match

**Example:** Logger `com.datadog.debugger.instrumentation.Foo`

Checks in order:
1. `datadog.slf4j.simpleLogger.log.com.datadog.debugger.instrumentation.Foo`
2. `datadog.slf4j.simpleLogger.log.com.datadog.debugger.instrumentation`
3. `datadog.slf4j.simpleLogger.log.com.datadog.debugger`
4. `datadog.slf4j.simpleLogger.log.com.datadog`
5. `datadog.slf4j.simpleLogger.log.com`
6. Default level

### Component Hierarchy

**Tracer Core:**
```
datadog.trace.core
├── datadog.trace.core.DDSpan
├── datadog.trace.core.PendingTrace
├── datadog.trace.core.DDTracer
└── datadog.trace.core.processor
```

**Dynamic Instrumentation:**
```
com.datadog.debugger
├── com.datadog.debugger.agent
├── com.datadog.debugger.instrumentation
├── com.datadog.debugger.probe
└── com.datadog.debugger.sink
```

**AppSec:**
```
com.datadog.appsec
├── com.datadog.appsec.gateway
├── com.datadog.appsec.event
└── com.datadog.appsec.powerwaf
```

**Profiling:**
```
com.datadog.profiling
├── com.datadog.profiling.controller
├── com.datadog.profiling.uploader
└── com.datadog.profiling.ddprof
```

**Remote Config:**
```
datadog.remoteconfig
├── datadog.remoteconfig.client
└── datadog.remoteconfig.poller
```

### Example Use Cases

**Scenario 1: Debug only DI**

```bash
java -Ddatadog.slf4j.simpleLogger.defaultLogLevel=info \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=debug \
     -javaagent:dd-java-agent.jar -jar app.jar
```

**Result:**
- All `com.datadog.debugger.*` → DEBUG
- Everything else → INFO

---

**Scenario 2: Debug tracer core + DI, silence profiling**

```bash
java -Ddd.trace.debug=true \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.profiling=warn \
     -javaagent:dd-java-agent.jar -jar app.jar
```

**Result:**
- Everything → DEBUG (from `dd.trace.debug=true`)
- `com.datadog.profiling.*` → WARN (overrides global)

---

**Scenario 3: Debug only span processing**

```bash
java -Ddatadog.slf4j.simpleLogger.defaultLogLevel=warn \
     -Ddatadog.slf4j.simpleLogger.log.datadog.trace.core.processor=debug \
     -javaagent:dd-java-agent.jar -jar app.jar
```

**Result:**
- `datadog.trace.core.processor.*` → DEBUG
- Everything else → WARN

---

**Scenario 4: Multiple component debugging**

```bash
java -Ddatadog.slf4j.simpleLogger.defaultLogLevel=info \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger=debug \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.appsec=debug \
     -Ddatadog.slf4j.simpleLogger.log.datadog.remoteconfig=debug \
     -javaagent:dd-java-agent.jar -jar app.jar
```

**Result:**
- DI, AppSec, RemoteConfig → DEBUG
- Everything else → INFO

---

**Scenario 5: Fine-grained sub-package**

```bash
java -Ddatadog.slf4j.simpleLogger.defaultLogLevel=warn \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.debugger.probe=debug \
     -javaagent:dd-java-agent.jar -jar app.jar
```

**Result:**
- Only `com.datadog.debugger.probe.*` → DEBUG
- `com.datadog.debugger.agent`, `com.datadog.debugger.sink`, etc. → WARN

### Discovering Package Names

**Method 1: Read the code**

Look at package declarations:
```java
// File: dd-java-agent/agent-debugger/src/main/.../ProbeConfiguration.java
package com.datadog.debugger.probe;

private static final Logger log = LoggerFactory.getLogger(ProbeConfiguration.class);
// Logger name: "com.datadog.debugger.probe.ProbeConfiguration"
```

**Method 2: Enable debug and observe**

```bash
java -Ddd.trace.debug=true -javaagent:dd-java-agent.jar -jar app.jar 2>&1 \
  | grep "^\[dd.trace" | awk '{print $5}' | cut -d']' -f1 | sort -u
```

Look for logger names like:
```
datadog.trace.core.DDTracer
com.datadog.debugger.agent.DebuggerAgent
com.datadog.appsec.AppSecSystem
```

**Method 3: Examine jar**

```bash
unzip -l dd-java-agent.jar | grep "\.class$" | awk '{print $4}' \
  | grep "com/datadog\|datadog/trace" | cut -d'/' -f1-4 | sort -u
```

### Strengths

- ✅ **Excellent component control**
- ✅ Standard Java package hierarchy
- ✅ SLF4J compatible
- ✅ Hierarchical (walks up package tree)
- ✅ Dynamic switching via Remote Config
- ✅ Production-safe

### Limitations

- ⚠️ Long system property names (verbose)
- ⚠️ Must know exact package names
- ⚠️ No wildcard support
- ⚠️ System properties vs env vars (less flexible)

---

## PHP (dd-trace-php): Global Level + Feature Flags

### Capability: ❌ Limited - Global Level Only

PHP does **not** provide per-component log level control. All components use the same global log level.

### Architecture

**File:** `/dd-trace-php/ext/logging.c`

```c
// Global log level applies to ALL components
bool ddtrace_alter_dd_trace_log_level(zval *old_value, zval *new_value, zend_string *new_str) {
    if (runtime_config_first_init ? get_DD_TRACE_DEBUG() : get_global_DD_TRACE_DEBUG()) {
        return true;  // DD_TRACE_DEBUG takes precedence
    }
    
    // Set global level for all components
    ddog_set_log_level(dd_zend_string_to_CharSlice(Z_STR_P(new_value)), ...);
    return true;
}
```

### Component Categorization (Internal)

**File:** `/dd-trace-php/components-rs/common.h`

```c
typedef enum ddog_Log {
    DDOG_LOG_ERROR = 1,
    DDOG_LOG_WARN = 2,
    DDOG_LOG_INFO = 3,
    DDOG_LOG_DEBUG = 4,
    DDOG_LOG_TRACE = 5,
    
    // Component-aware levels (bitfield)
    DDOG_LOG_STARTUP = (3 | (2 << 4)),        // INFO + startup component
    DDOG_LOG_SPAN = (4 | (3 << 4)),           // DEBUG + span component
    DDOG_LOG_SPAN_TRACE = (5 | (3 << 4)),     // TRACE + span component
    DDOG_LOG_HOOK_TRACE = (5 | (4 << 4)),     // TRACE + hook component
} ddog_Log;
```

**Component IDs:**
- **0**: DEFAULT
- **2**: STARTUP
- **3**: SPAN
- **4**: HOOK

**However:** These are **internal categorizations** not exposed to users.

### Workaround: Feature-Level Control

**Available control:**

```bash
# Global log level (applies to ALL components)
export DD_TRACE_LOG_LEVEL=debug  # error, warn, info, debug, trace

# Feature flags (enable/disable entire features)
export DD_TRACE_STARTUP_LOGS=0                 # Disable startup logs
export DD_TRACE_ONCE_LOGS=0                    # Allow repeated logs
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=0    # Disable DI entirely
export DD_APPSEC_ENABLED=0                     # Disable AppSec entirely
```

### Example Use Cases

**Scenario 1: Tracer logs, no DI**

```bash
export DD_TRACE_LOG_LEVEL=debug
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=0

php app.php
```

**Result:**
- Tracer logs → DEBUG
- DI disabled (no logs)

---

**Scenario 2: Minimal startup, error-level only**

```bash
export DD_TRACE_LOG_LEVEL=error
export DD_TRACE_STARTUP_LOGS=0

php app.php
```

**Result:**
- No startup JSON
- Only errors logged

---

**Scenario 3: Debug everything except disable once-only logs**

```bash
export DD_TRACE_DEBUG=1
export DD_TRACE_ONCE_LOGS=0

php app.php
```

**Result:**
- All components → DEBUG/TRACE
- Repeated logs allowed

### Limitations

- ❌ **No per-component log levels**
- ❌ Cannot debug one component while silencing others
- ❌ Feature flags are all-or-nothing (enable/disable entire feature)
- ❌ No fine-grained control

### Strengths

- ✅ Simple configuration
- ✅ Easy to understand
- ✅ Fast (no complex filtering)
- ✅ Feature flags provide some control

---

## .NET (dd-trace-dotnet): Per-Type Logger Retrieval

### Capability: ⚠️ Partial - Per-Type Loggers

.NET uses **per-type logger instances** but lacks environment variable-based component control.

### Architecture

**File:** `/dd-trace-dotnet/tracer/src/Datadog.Trace/Logging/Internal/DatadogLogging.cs` (lines 54-62)

```csharp
public static IDatadogLogger GetLoggerFor(Type classType)
{
    // Each type gets its own logger instance
    if (SharedLogger.IsEnabled(LogEventLevel.Debug))
    {
        SharedLogger.Debug("Logger retrieved for: {AssemblyQualifiedName}", 
                          classType.AssemblyQualifiedName);
    }
    return SharedLogger;
}
```

**Usage in components:**

```csharp
// File: Datadog.Trace/Debugger/DynamicInstrumentation.cs
private static readonly IDatadogLogger Log = DatadogLogging.GetLoggerFor<DynamicInstrumentation>();

// File: Datadog.Trace/Span.cs
private static readonly IDatadogLogger Log = DatadogLogging.GetLoggerFor(typeof(Span));

// File: Datadog.Trace/TracerManager.cs
private static readonly IDatadogLogger Log = DatadogLogging.GetLoggerFor<TracerManager>();
```

### Component Organization

**Tracer Core:**
- `Datadog.Trace.Span`
- `Datadog.Trace.Tracer`
- `Datadog.Trace.TracerManager`
- `Datadog.Trace.Agent.AgentWriter`

**Dynamic Instrumentation:**
- `Datadog.Trace.Debugger.DynamicInstrumentation`
- `Datadog.Trace.Debugger.ProbeStatus`
- `Datadog.Trace.Debugger.Upload.SnapshotUploadApi`

**AppSec:**
- `Datadog.Trace.AppSec.AppSecSystem`
- `Datadog.Trace.AppSec.Waf.WafLibraryInvoker`

**Profiling:**
- `Datadog.Trace.Profiler.ProfilerManager`

### Workaround: Serilog Filtering

**Method 1: Programmatic filter**

```csharp
using Serilog;
using Serilog.Events;

var loggerConfiguration = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Datadog.Trace.Debugger", LogEventLevel.Debug)
    .MinimumLevel.Override("Datadog.Trace.AppSec", LogEventLevel.Warning);
```

**Method 2: Environment variable + code**

```csharp
// Read custom env var
var diLogLevel = Environment.GetEnvironmentVariable("DD_DEBUGGER_LOG_LEVEL") ?? "Information";

var config = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Datadog.Trace.Debugger", 
                          Enum.Parse<LogEventLevel>(diLogLevel));
```

### Example Use Cases

**Scenario 1: Debug only DI (requires code)**

```csharp
var config = new LoggerConfiguration()
    .MinimumLevel.Warning()
    .MinimumLevel.Override("Datadog.Trace.Debugger", LogEventLevel.Debug);
```

**Scenario 2: Use environment variable (custom implementation)**

```bash
export DD_TRACE_LOG_LEVEL=Warning
export DD_DEBUGGER_LOG_LEVEL=Debug
export DD_APPSEC_LOG_LEVEL=Information
```

```csharp
// Custom configuration loader
var config = new LoggerConfiguration()
    .MinimumLevel.Is(ParseLogLevel("DD_TRACE_LOG_LEVEL", LogEventLevel.Information));

foreach (var component in new[] { "Debugger", "AppSec", "Profiler" })
{
    var envVar = $"DD_{component.ToUpper()}_LOG_LEVEL";
    if (Environment.GetEnvironmentVariable(envVar) is string level)
    {
        config.MinimumLevel.Override(
            $"Datadog.Trace.{component}", 
            Enum.Parse<LogEventLevel>(level)
        );
    }
}
```

### Limitations

- ❌ **No built-in env var component control**
- ❌ Requires code changes or custom implementation
- ⚠️ Serilog filtering available but not pre-configured
- ⚠️ Per-type loggers exist but all share same level

### Strengths

- ✅ Per-type logger infrastructure exists
- ✅ Serilog supports filtering (just not configured)
- ✅ Clean component organization
- ✅ Could be added in future versions

---

## Go (dd-trace-go): Feature Flags Only

### Capability: ❌ None - Global Level Only

Go does **not** provide per-component log level control. All components use the same global log level.

### Architecture

**File:** `/dd-trace-go/internal/log/log.go`

```go
var (
    levelThreshold = LevelWarn  // Global threshold
    logger         Logger       // Single logger instance
)

// SetLevel sets the global log level threshold
func SetLevel(lvl Level) {
    levelThreshold = lvl
}

// Debug logs only if global level is Debug
func Debug(msg string) {
    if levelThreshold > LevelDebug {
        return
    }
    logger.Log(msg)
}
```

### Workaround: Feature-Specific Flags

```bash
# Global debug
export DD_TRACE_DEBUG=true

# Abandoned spans (separate feature)
export DD_TRACE_DEBUG_ABANDONED_SPANS=true
export DD_TRACE_ABANDONED_SPAN_TIMEOUT=10m

# Dynamic Instrumentation (enable/disable)
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=true
```

### Example Use Cases

**Scenario 1: Debug everything**

```bash
export DD_TRACE_DEBUG=true
```

**Result:** All components → DEBUG

---

**Scenario 2: Debug + abandoned span detection**

```bash
export DD_TRACE_DEBUG=true
export DD_TRACE_DEBUG_ABANDONED_SPANS=true
```

**Result:**
- All components → DEBUG
- Abandoned spans logged every 1 minute

---

**Scenario 3: Warnings only (default)**

```bash
# No env vars needed
go run main.go
```

**Result:** All components → WARN (default)

### Limitations

- ❌ **No per-component log levels**
- ❌ Cannot debug one component while silencing others
- ⚠️ Feature flags provide limited control (enable/disable features, not log levels)

### Strengths

- ✅ Simple global configuration
- ✅ Fast (no filtering overhead)
- ✅ Abandoned spans as separate feature flag

---

## JavaScript (dd-trace-js): Diagnostic Channels

### Capability: ⚠️ Partial - Diagnostic Channel Subscriptions

JavaScript provides **diagnostic channels** for subscribing to specific log levels.

### Architecture

**File:** `/dd-trace-js/packages/dd-trace/src/log/channels.js`

```javascript
const dc = require('diagnostics_channel')

const logChannels = {
  trace: dc.channel('datadog:log:trace'),
  debug: dc.channel('datadog:log:debug'),
  info: dc.channel('datadog:log:info'),
  warn: dc.channel('datadog:log:warn'),
  error: dc.channel('datadog:log:error')
}
```

**File:** `/dd-trace-js/packages/dd-trace/src/log/index.js`

```javascript
function log(level, message) {
  // Global level check
  if (level < currentLogLevel) {
    return
  }
  
  // Publish to diagnostic channel
  const channel = logChannels[levelToString(level)]
  if (channel.hasSubscribers) {
    channel.publish({ message, level })
  }
  
  // Also write to configured logger
  logger.write(level, message)
}
```

### Component Organization

**Logger locations:**
- Main tracer: `/packages/dd-trace/src/log/index.js`
- Debugger client: `/packages/dd-trace/src/debugger/devtools_client/log.js`
- Worker thread: Separate logger instance
- Guardrails: `/packages/dd-trace/src/guardrails/log.js` (pre-init)

### Using Diagnostic Channels

**Method 1: Subscribe to specific channels**

```javascript
const dc = require('diagnostics_channel')

// Only listen to debug messages
dc.subscribe('datadog:log:debug', (msg) => {
  console.log('[DEBUG]', msg.message)
})

// Only listen to error messages
dc.subscribe('datadog:log:error', (msg) => {
  console.error('[ERROR]', msg.message)
})
```

**Method 2: Custom logger with filtering**

```javascript
const tracer = require('dd-trace')

const customLogger = {
  debug: (msg) => {
    // Filter: Only log messages containing 'debugger'
    if (msg.includes('debugger')) {
      console.log('[DEBUG]', msg)
    }
  },
  info: (msg) => console.log('[INFO]', msg),
  warn: (msg) => console.warn('[WARN]', msg),
  error: (msg) => console.error('[ERROR]', msg)
}

tracer.init({
  logger: customLogger,
  logLevel: 'debug'
})
```

**Method 3: Winston with namespace filtering**

```javascript
const winston = require('winston')
const tracer = require('dd-trace')

const logger = winston.createLogger({
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.label({ label: 'dd-trace' }),
        winston.format.timestamp(),
        winston.format.printf(({ level, message, label, timestamp }) => {
          // Filter: Only show debugger-related logs
          if (message.includes('debugger') || message.includes('probe')) {
            return `${timestamp} [${label}] ${level}: ${message}`
          }
          return null
        })
      )
    })
  ]
})

tracer.init({
  logger: logger,
  logLevel: 'debug'
})
```

### Example Use Cases

**Scenario 1: Subscribe to debug channel only**

```javascript
const dc = require('diagnostics_channel')

dc.subscribe('datadog:log:debug', (msg) => {
  console.log(msg.message)
})

// Set tracer to debug level
require('dd-trace').init({ logLevel: 'debug' })
```

**Result:** Only debug messages logged via channel

---

**Scenario 2: Custom logger with component filtering**

```javascript
const tracer = require('dd-trace')

tracer.init({
  logLevel: 'debug',
  logger: {
    debug: (msg) => {
      // Only log debugger-related messages
      if (msg.match(/debugger|probe|snapshot/i)) {
        console.log('[DEBUG]', msg)
      }
    },
    info: () => {},   // Suppress info
    warn: (msg) => console.warn('[WARN]', msg),
    error: (msg) => console.error('[ERROR]', msg)
  }
})
```

**Result:** Only debugger-related debug messages, plus all warnings/errors

---

**Scenario 3: Winston with namespace-like filtering**

```javascript
const winston = require('winston')
const tracer = require('dd-trace')

const filterFormat = winston.format((info) => {
  const msg = info.message || ''
  
  // Allow: debugger, probe, snapshot, instrumentation
  if (msg.match(/debugger|probe|snapshot|instrumentation/i)) {
    return info
  }
  
  // Block everything else at debug level
  if (info.level === 'debug') {
    return false
  }
  
  // Allow warn/error from all components
  return info
})

const logger = winston.createLogger({
  format: winston.format.combine(
    filterFormat(),
    winston.format.simple()
  ),
  transports: [new winston.transports.Console()]
})

tracer.init({
  logger: logger,
  logLevel: 'debug'
})
```

**Result:** Debug logging only for DI-related messages

### Limitations

- ⚠️ **No built-in env var component control**
- ⚠️ Requires programmatic configuration
- ⚠️ String matching on log messages (brittle)
- ⚠️ Diagnostic channels don't distinguish components

### Strengths

- ✅ Diagnostic channels provide flexible subscription
- ✅ Pluggable logger interface
- ✅ Winston/Bunyan/custom logger support
- ✅ Can implement custom filtering logic

---

## Comparison Summary

### Component Control Ranking

1. **🥇 Python** - Hierarchical env vars, prefix trie, production-ready
2. **🥈 Java** - Package hierarchy, system properties, robust
3. **🥉 JavaScript** - Diagnostic channels, pluggable loggers, requires code
4. **.NET** - Per-type loggers exist, no env var control
5. **PHP** - Global only, feature flags for enable/disable
6. **Go** - Global only, feature flags for abandoned spans
7. **Ruby** - Global only, requires custom filter implementation

### Use Case Recommendations

| Use Case | Best Tracer | Why |
|----------|-------------|-----|
| **Production DI debugging** | Python, Java | Env var control, no code changes |
| **Multi-component debugging** | Python | Easy hierarchical env vars |
| **SLF4J integration** | Java | Standard Java logging |
| **Custom logger integration** | JavaScript | Pluggable interface |
| **Simple global control** | PHP, Go | Fast, no filtering overhead |
| **Enterprise .NET** | .NET | Serilog filtering (with code) |
| **Flexible programmatic control** | Ruby, JavaScript | Full language power for custom logic |

---

## Implementation Complexity

### Adding Component Control (Hypothetical)

**Easy to add:**
- **.NET** - Serilog already supports namespace filtering, just need env var parsing
- **JavaScript** - Diagnostic channels exist, need namespace tagging

**Moderate effort:**
- **Go** - Add per-package level map, update log calls with package identifier
- **PHP** - Extend bitfield system, add env var parsing for component IDs

**Significant refactoring:**
- **Ruby** - Need to split shared logger into per-component instances

---

## Best Practices

### When to Use Component Control

✅ **Use component control when:**
- Debugging specific feature in production
- Investigating component interaction issues
- Reducing log volume in high-throughput systems
- Testing new component without noise from others

❌ **Don't use component control when:**
- Initial troubleshooting (use global debug first)
- Unknown issue (need visibility across all components)
- Development environment (log volume not a concern)

### Recommended Configurations

**Production DI debugging (Python):**
```bash
export DD_TRACE_LOG_LEVEL=WARNING
export _DD_DEBUGGING_LOG_LEVEL=DEBUG
export DD_TRACE_LOGGING_RATE=60
```

**Production AppSec debugging (Java):**
```bash
java -Ddatadog.slf4j.simpleLogger.defaultLogLevel=warn \
     -Ddatadog.slf4j.simpleLogger.log.com.datadog.appsec=debug \
     -Ddd.log.format.json=true
```

**Development all components (Go):**
```bash
export DD_TRACE_DEBUG=true
export DD_LOGGING_RATE=0
```

---

## Conclusion

**Component control is critical for production debugging.** Python and Java provide the best solutions with environment variable-based hierarchical control, making them ideal for enterprise production environments where targeted debugging is essential without full restarts or code changes.

**Tracers without component control** (Ruby, PHP, Go) should consider adding this feature for better production operability.
