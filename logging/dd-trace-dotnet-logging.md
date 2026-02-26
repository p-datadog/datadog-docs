# dd-trace-dotnet Logging Configuration Guide

## Overview

This guide explains how to configure logging for Datadog's .NET tracer (dd-trace-dotnet) to control tracer, Dynamic Instrumentation (DI), and component logging output.

## Key Characteristics

**dd-trace-dotnet** logging features:
- Uses **Serilog framework** (vendored) for structured logging
- Logs **full span details** on start and finish in debug mode
- **JSON startup diagnostic logs** with complete configuration
- **Size-based log rotation** (default: 10MB files)
- **Automatic log file cleanup** (default: 32 days retention)
- **Telemetry integration** - tracks log creation metrics
- **Rate limiting** per message location (file:line)
- **AppDomain enrichment** for multi-domain scenarios

---

## How DD_TRACE_DEBUG Works

### Configuration Location

**Environment variable:** `DD_TRACE_DEBUG`  
**Configuration key:** `ConfigurationKeys.DebugEnabled`

**File:** `/tracer/src/Datadog.Trace.ClrProfiler.Managed.Loader/StartupLogger.cs` (lines 22-45)

### Behavior When Enabled

1. **Logger level** set to `LogEventLevel.Debug`
2. **Startup logging** includes AppDomain information:
   - Format: `[timestamp|AppDomainId|AppDomainName|IsDefaultAppDomain] message`
3. **Span logging** outputs full details on start and close
4. **Type loading** tracked with debug messages
5. **Native tracer** receives debug flag via LibDatadog

**Example startup output:**
```
[2024-02-25 10:30:45.123 -05:00|1|MyApp|True] Datadog Tracer initialization starting
```

---

## What Gets Logged When Debug Mode is Enabled

### Span Creation

**File:** `/tracer/src/Datadog.Trace/Span.cs`

```csharp
Log.Debug(
    "Span started: [s_id: {SpanId}, p_id: {ParentId}, t_id: {TraceId}] " +
    "with Tags: [{Tags}], Tags Type: [{TagsType}])",
    SpanId, Context.ParentId, Context.TraceId, Tags, Tags?.GetType());
```

### Span Closure

```csharp
Log.Debug(
    "Span closed: [s_id: {SpanId}, p_id: {ParentId}, t_id: {TraceId}] " +
    "for (Service: {ServiceName}, Resource: {ResourceName}, Operation: {OperationName}, " +
    "Tags: [{Tags}])\nDetails:{ToString}",
    SpanId, Context.ParentId, Context.TraceId, ServiceName, ResourceName, 
    OperationName, Tags, ToString());
```

**Output includes:**
- Span ID, Parent ID, Trace ID
- Service name, Resource name, Operation name
- All tags (metadata)
- All metrics
- Complete span string representation

### Startup Diagnostic JSON

**File:** `/tracer/src/Datadog.Trace/TracerManager.cs` (lines 330-500+)

**Content:**
```json
{
  "os_name": "Windows",
  "os_version": "10.0.19045",
  "platform": "x64",
  "architecture": "X64",
  "tracer_version": "2.45.0",
  "native_tracer_version": "2.45.0",
  "lang": ".NET",
  "lang_version": "6.0.25",
  "enabled": true,
  "service": "my-dotnet-app",
  "agent_url": "http://localhost:8126",
  "debug": true,
  "analytics_enabled": false,
  "sample_rate": "1.0",
  "sampling_rules": [],
  "tags": {},
  "disabled_integrations": [],
  "health_check_enabled": true,
  "runtime_metrics_enabled": true,
  "transport": "Default"
}
```

### Logger Type Retrieval

**File:** `/tracer/src/Datadog.Trace/Logging/Internal/DatadogLogging.cs` (lines 54-62)

```csharp
public static IDatadogLogger GetLoggerFor(Type classType)
{
    if (SharedLogger.IsEnabled(LogEventLevel.Debug))
    {
        SharedLogger.Debug("Logger retrieved for: {AssemblyQualifiedName}", 
                          classType.AssemblyQualifiedName);
    }
    return SharedLogger;
}
```

Tracks which types initialize loggers and when.

---

## Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| **DD_TRACE_DEBUG** | `false` | Enable debug mode logging |
| **DD_TRACE_LOG_DIRECTORY** | Windows: `%ProgramData%\Datadog .NET Tracer\logs\`<br>Linux: `/var/log/datadog/dotnet/` | Log file directory |
| **DD_TRACE_LOG_PATH** | N/A | Individual log file path (deprecated) |
| **DD_TRACE_LOG_SINKS** | `"file"` | Log output destinations |
| **DD_TRACE_LOGGING_RATE** | `0` (disabled) | Rate limiting in seconds between identical messages |
| **DD_TRACE_LOGFILE_RETENTION_DAYS** | `32` | Auto-cleanup threshold (days) |
| **DD_MAX_LOGFILE_SIZE** | `10 MB` | Max size before rolling to new file |
| **DD_TRACE_STARTUP_LOGS** | `true` | Enable startup diagnostic JSON logs |

**Configuration file:** `/tracer/src/Datadog.Trace/Configuration/supported-configurations-docs.yaml`

---

## Log Levels

### Serilog Log Levels

```csharp
public enum LogEventLevel
{
    Verbose = 0,  // Not used in tracer
    Debug = 1,    // Enabled by DD_TRACE_DEBUG
    Information = 2,  // Default operational level
    Warning = 3,
    Error = 4,
    Fatal = 5     // Critical errors only
}
```

### Setting Log Level

**Via environment:**
```bash
export DD_TRACE_DEBUG=true  # Sets to Debug
```

**Programmatically:**
```csharp
GlobalSettings.SetDebugEnabled(true);
```

**Via LibDatadog:**
```csharp
LibDatadog.Logging.Logger.Instance.SetLogLevel(debugEnabled: true);
```

---

## Rate Limiting

### Log Rate Limiter

**File:** `/tracer/src/Datadog.Trace/Logging/Internal/LogRateLimiter.cs`

**Key:** Source file path + line number

**Configuration:**
```bash
export DD_TRACE_LOGGING_RATE=60  # Log once per 60 seconds per location
```

**Behavior:**
- Time-bucketed sampling based on `DD_TRACE_LOGGING_RATE`
- Tracks skipped count for each location
- Output includes: `"{message}, {SkipCount} additional messages skipped"`

**Example:**
```
Connection failed, 5 additional messages skipped
```

### Probe Rate Limiting (Dynamic Instrumentation)

**File:** `/tracer/src/Datadog.Trace/Debugger/RateLimiting/ProbeRateLimiter.cs`

```csharp
public void SetRate(string probeId, int samplesPerSecond)
{
    // Log probes with snapshots: 1 sample/second
    // Log probes without snapshots: 5000 samples/second
    var adaptiveSampler = CreateSampler(samplesPerSecond);
    _samplers.TryAdd(probeId, adaptiveSampler);
}
```

**Defaults:**
- **With snapshots:** 1 sample per second
- **Without snapshots:** 5000 samples per second

---

## Dynamic Instrumentation Logging

### Probe Configuration

**File:** `/tracer/src/Datadog.Trace/Debugger/DynamicInstrumentation.cs`

**Log messages:**
```csharp
Log.Information<int>(
    "Dynamic Instrumentation.InstrumentProbes: Request to instrument {Count} probes definitions", 
    addedProbes.Count);

Log.Information(
    "Finished resolving line probe for ProbeID {ProbeID}. Result was '{Status}'. Message was: '{Message}'", 
    addedProbe.Id, status, message);

Log.Warning(
    "ProbeID {ProbeID} error resolving live. Error: {Error}", 
    addedProbe.Id, message);
```

### Snapshot Upload

**File:** `/tracer/src/Datadog.Trace/Debugger/Upload/SnapshotUploadApi.cs`

```csharp
Log.Debug("SnapshotUploadApi: Updated endpoint to {Endpoint}", Endpoint);
Log.Warning("SnapshotUploadApi: No discovery service or static endpoint available");
Log.Debug("SnapshotUploadApi: Sending snapshots to {Uri}", uri);
```

**What gets logged:**
- Probe instrumentation status (bound/unbound/error)
- Successful vs failed probe resolutions
- Snapshot upload endpoint changes
- Upload success/failure

**What does NOT get logged:**
- Snapshot data contents
- Captured variable values
- Stack traces (sent to agent, not logged)

---

## Log File Management

### File Naming Pattern

```
dotnet-tracer-managed-{ProcessName}-{ProcessId}.log
```

**Example:**
```
dotnet-tracer-managed-MyApp-12345.log
```

### Rolling Policy

**Trigger:** File size exceeds `DD_MAX_LOGFILE_SIZE` (default: 10MB)

**Behavior:**
- Creates new file with incremented suffix
- Old files retained based on `DD_TRACE_LOGFILE_RETENTION_DAYS` (default: 32 days)

**Auto-cleanup targets:**
- `dotnet-tracer-*.log`
- `dotnet-native-loader-*.log`
- `DD-DotNet-Profiler-Native-*.log`

**Location:** `/tracer/src/Datadog.Trace/Logging/Internal/DatadogLogging.cs` (lines 90-136)

---

## Serilog Configuration

### Output Template

**File:** `/tracer/src/Datadog.Trace/Logging/Internal/DatadogSerilogLogger.cs`

```
{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj} {Exception} {Properties}{NewLine}
```

**Example output:**
```
2024-02-25 10:30:45.123 -05:00 [DBG] Span started: [s_id: 12345, p_id: 0, t_id: 67890] with Tags: [...]
```

### Automatic Enrichment

**Added properties:**
- **MachineName:** `Environment.MachineName`
- **Process:** `[PID ProcessName]`
- **AppDomain:** `[AppDomainId AppDomainName]`
- **AssemblyLoadContext:** (on .NET Core)
- **TracerVersion:** Tracer version string

**File:** `/tracer/src/Datadog.Trace/Logging/Internal/DatadogSerilogLogger.cs`

---

## Telemetry Integration

### Log Metrics

**File:** `/tracer/src/Datadog.Trace/Logging/Internal/DatadogSerilogLogger.cs` (lines 175-215)

```csharp
private void WriteIfNotRateLimited(LogEventLevel level, Exception? exception, 
                                   string messageTemplate, object?[] args, 
                                   int sourceLine, string sourceFile, bool skipTelemetry)
{
    TelemetryFactory.Metrics.RecordCountLogCreated(LevelToTag(level));
    if (_rateLimiter.ShouldLog(sourceFile, sourceLine, out var skipCount))
    {
        // Log includes skip count if messages were rate-limited
    }
}
```

**Tracked metrics:**
- Log creation count per level (Debug/Info/Warning/Error)
- Rate-limited message skip counts

### Telemetry Suppression

**Marker property:** `SkipTelemetryProperty`

Usage:
```csharp
log.Error("Message")
   .Property(DatadogSerilogLogger.SkipTelemetryProperty, true);
```

### Error Redaction for Telemetry

**File:** `/tracer/src/Datadog.Trace/Logging/Internal/DatadogLoggingFactory.cs` (lines 98-107)

```csharp
if (config.ErrorLogging is { } telemetry)
{
    loggerConfiguration
       .WriteTo.Logger(
            lc => lc
                 .MinimumLevel.Error()
                 .Filter.ByExcluding(Matching.WithProperty(SkipTelemetryProperty))
                 .WriteTo.Sink(new RedactedErrorLogSink(telemetry.Collector)));
}
```

Error logs sent to telemetry are redacted via `RemovePropertyEnricher`.

---

## Configuration Scenarios

### Scenario 1: Development (Full Debug)

```bash
export DD_TRACE_DEBUG=true
export DD_TRACE_LOGGING_RATE=0  # No rate limiting
export DD_TRACE_STARTUP_LOGS=true
```

**What you get:**
- Full span details on start/finish
- AppDomain-enriched logs
- Startup diagnostic JSON
- All debug messages
- No rate limiting

---

### Scenario 2: Production (Conservative)

```bash
export DD_TRACE_LOG_LEVEL=Warning
export DD_TRACE_LOGGING_RATE=60
export DD_MAX_LOGFILE_SIZE=10485760  # 10MB
export DD_TRACE_LOGFILE_RETENTION_DAYS=7
```

**What you get:**
- Warnings and errors only
- Rate limiting: 1 log per minute per location
- 10MB log files
- 7-day retention
- Automatic cleanup

---

### Scenario 3: DI Debugging

```bash
export DD_TRACE_DEBUG=true
export DD_DYNAMIC_INSTRUMENTATION_ENABLED=true
```

**What you get:**
- DI probe instrumentation logs
- Probe resolution status (bound/unbound/error)
- Snapshot upload logs
- Full span details

---

### Scenario 4: Minimal Logging

```bash
export DD_TRACE_LOG_LEVEL=Error
export DD_TRACE_STARTUP_LOGS=false
```

**What you get:**
- Errors only
- No startup JSON
- No debug/info/warning messages

---

## Best Practices

### Development

```bash
export DD_TRACE_DEBUG=true
export DD_TRACE_LOGGING_RATE=0
```

**Benefits:**
- Full visibility into tracer operations
- Span details for debugging
- No rate limiting for complete output

---

### Production

```bash
export DD_TRACE_LOG_LEVEL=Warning
export DD_TRACE_LOGGING_RATE=60
export DD_MAX_LOGFILE_SIZE=10485760
export DD_TRACE_LOGFILE_RETENTION_DAYS=7
export DD_TRACE_LOG_DIRECTORY=/var/log/datadog/dotnet/
```

**Benefits:**
- Minimal noise (warnings and errors)
- Rate limiting prevents log flooding
- Automatic log rotation
- 7-day retention prevents disk fill
- Centralized log directory

---

### Docker/Container Environments

```bash
export DD_TRACE_LOG_LEVEL=Information
export DD_TRACE_STARTUP_LOGS=true
export DD_TRACE_LOG_DIRECTORY=/app/logs/
```

**Benefits:**
- Startup JSON for configuration audit
- Info level for operational visibility
- Container-local log directory

---

## Limitations and Gotchas

### No Per-Component Env Var Control

**Limitation:** Cannot set different log levels for different components via environment variables.

**Workaround:** Use Serilog filtering in code or custom logger configuration.

### No Remote Config Support

**Limitation:** Cannot dynamically change log level at runtime without restart (unlike Java).

**Workaround:** Plan log levels carefully; use container orchestration for quick restarts.

### Large Log Files

**Issue:** Debug mode can generate large log files quickly (10MB+ per hour in busy apps).

**Solution:** Use smaller `DD_MAX_LOGFILE_SIZE` and shorter `DD_TRACE_LOGFILE_RETENTION_DAYS` in debug mode.

### AppDomain Overhead

**Issue:** AppDomain enrichment adds overhead to each log message.

**Impact:** Minimal (~microseconds per log), but visible in high-frequency logging.

### Telemetry Overhead

**Issue:** Telemetry metrics for every log message.

**Impact:** Minimal, but can be suppressed with `SkipTelemetryProperty` if needed.

---

## Verbosity Analysis

### Per Request (3 spans, debug mode)

**Span lifecycle:**
- Root span start: ~2-3 lines
- Child span starts: ~2-3 lines each
- Span closes: ~3-5 lines each (includes full details)

**Total:** ~10-30 lines per trace

**Startup (one-time):**
- Startup JSON: 1 line (~500-1000 characters)
- AppDomain info: ~2-5 lines
- Logger initialization: ~5-10 lines

**First request total:** ~30-50 lines  
**Subsequent requests:** ~10-30 lines

---

## Code References

### Core Logging
- Logger factory: `/tracer/src/Datadog.Trace/Logging/Internal/DatadogLoggingFactory.cs`
- Serilog wrapper: `/tracer/src/Datadog.Trace/Logging/Internal/DatadogSerilogLogger.cs`
- Rate limiter: `/tracer/src/Datadog.Trace/Logging/Internal/LogRateLimiter.cs`
- DatadogLogging: `/tracer/src/Datadog.Trace/Logging/Internal/DatadogLogging.cs`

### Span Logging
- Span class: `/tracer/src/Datadog.Trace/Span.cs`

### Startup Logging
- StartupLogger: `/tracer/src/Datadog.Tracer.Native/StartupLogger.cs`
- TracerManager: `/tracer/src/Datadog.Trace/TracerManager.cs` (lines 330-500+)

### Dynamic Instrumentation
- DynamicInstrumentation: `/tracer/src/Datadog.Trace/Debugger/DynamicInstrumentation.cs`
- ProbeRateLimiter: `/tracer/src/Datadog.Trace/Debugger/RateLimiting/ProbeRateLimiter.cs`
- SnapshotUploadApi: `/tracer/src/Datadog.Trace/Debugger/Upload/SnapshotUploadApi.cs`

### Configuration
- Config keys: `/tracer/src/Datadog.Trace/Generated/net6.0/ConfigurationKeys.g.cs`
- Config docs: `/tracer/src/Datadog.Trace/Configuration/supported-configurations-docs.yaml`

---

## Summary

**dd-trace-dotnet logging is designed for enterprise .NET applications:**

✅ **Structured logging** - Serilog framework with enrichment  
✅ **Full span details** - Complete span logging in debug mode  
✅ **Automatic rotation** - Size-based (10MB default)  
✅ **Rate limiting** - Per-message location with skip counts  
✅ **Telemetry** - Integrated metrics for log creation  
✅ **Startup JSON** - Comprehensive configuration audit  
✅ **AppDomain enrichment** - Multi-domain scenario support  

❌ **No remote config** - Requires restart for level changes  
❌ **No per-component env vars** - Global level only  
❌ **Large files in debug** - Can grow quickly  

**When to use .NET tracer logging:**
- ✅ Startup configuration auditing
- ✅ Full span inspection for debugging
- ✅ Multi-AppDomain scenarios
- ✅ Enterprise production environments with structured logging
- ✅ Telemetry-integrated observability

**When NOT to use:**
- ❌ High-frequency logging (use sampling)
- ❌ Dynamic log level changes (no remote config)
- ❌ Storage-constrained environments (large files)

---

## Quick Reference

### Enable Debug
```bash
export DD_TRACE_DEBUG=true
```

### Set Log Directory
```bash
export DD_TRACE_LOG_DIRECTORY=/var/log/datadog/dotnet/
```

### Configure Rate Limiting
```bash
export DD_TRACE_LOGGING_RATE=60  # Seconds between logs
```

### Set Max File Size
```bash
export DD_MAX_LOGFILE_SIZE=10485760  # 10MB in bytes
```

### Configure Retention
```bash
export DD_TRACE_LOGFILE_RETENTION_DAYS=7
```

### Production Config
```bash
export DD_TRACE_LOG_LEVEL=Warning
export DD_TRACE_LOGGING_RATE=60
export DD_MAX_LOGFILE_SIZE=10485760
export DD_TRACE_LOGFILE_RETENTION_DAYS=7
export DD_TRACE_STARTUP_LOGS=true
```
