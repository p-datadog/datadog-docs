# DTR Component Logging Summary

Complete analysis of logging output from all major dd-trace-rb components.

## Documents Created

1. **dtr-logging.md** - Tracer and DI logging configuration (already in datadog-docs/)
2. **This document** - Profiling, AppSec, Core, and Data Streams logging analysis

## Component Analysis

### 1. PROFILING (~28 log statements)

**Most verbose component during initialization**

**Key Events:**
- Ruby version compatibility checks (GC/allocation/heap profiling)
- Gem compatibility detection (MySQL2, Passenger, Rugged)
- "No signals" workaround warnings
- Thread lifecycle operations
- SIGPROF signal conflicts

**Examples:**
```
[WARN] Allocation profiling not supported in Ruby 3.2.0-3.2.2 due to VM bug
[WARN] Enabling "no signals" workaround for mysql2 gem - profiling quality reduced
[DEBUG] Starting thread for: CpuAndWallTimeWorker
[DEBUG] Enabled heap profiling: heap_sample_rate=512
```

### 2. APPSEC (~10 log statements)

**Security and configuration validation**

**Key Events:**
- Configuration validation (modes, settings)
- WAF security engine lifecycle
- Rails middleware introspection
- Request processing errors

**Examples:**
```
[ERROR] AppSec security engine failed to initialize
[WARN] appsec.track_user_events.mode value not supported
[DEBUG] Rails middlewares: [ActionDispatch::HostAuthorization, ...]
```

### 3. CORE (~24 log statements)

**Infrastructure and feature availability**

**Components:**
- Remote Configuration client
- Telemetry
- Crash Tracking
- Feature availability checks

**Examples:**
```
[DEBUG] new remote configuration client: <uuid> products: appsec, asm_data
[ERROR] remote worker sync error: connection timeout
[WARN] Missing ld_library_path; cannot enable crash tracking
[INFO] {"stat":"requests.count","type":"count","value":100}
```

### 4. DATA_STREAMS (~2 log statements)

**Minimal operational logging**

**Examples:**
```
[DEBUG] Failed to flush DSM stats to agent: IOError
[DEBUG] DSM stats sent to agent: ok=true
```

---

## Summary Table

| Component | Log Levels | Volume | Primary Purpose |
|-----------|------------|--------|-----------------|
| Tracing | DEBUG | Very High (50-150 lines/trace) | Full span details |
| Profiling | DEBUG/WARN | High (startup) | Compatibility diagnostics |
| Core | DEBUG/WARN/ERROR | Medium | Client operations |
| AppSec | DEBUG/WARN/ERROR | Medium | Security engine |
| DI | DEBUG (trace) | Variable | Probe lifecycle |
| Data Streams | DEBUG | Very Low | Operation results |

---

## Configuration

### DD_TRACE_DEBUG Behavior

**Enables:**
- Logger level set to DEBUG for ALL components
- Caller stack traces in all log messages
- Tracer: Full span output via `pretty_inspect`
- DI: Verbose trace logs (`trace_logging = true`)

**Does NOT provide:**
- Component-specific filtering flags
- Per-component verbosity controls

### Filtering Strategy

Custom logger with prefix-based filtering required:

```ruby
# Filter out profiling logs
return if msg.to_s.match?(/profiling|CpuAndWallTimeWorker/)

# Filter out AppSec logs
return if msg.to_s.include?('AppSec:')

# Filter out remote config logs
return if msg.to_s.match?(/remote worker|remote configuration/)
```

---

## Logging Patterns

- **Lazy evaluation**: `logger.debug { }` blocks avoid string construction overhead
- **Error context**: Format includes `Cause: Exception Location: file.rb:line`
- **Structured data**: Metrics use JSON format
- **Consistent prefixes**: `"di:"`, `"AppSec:"`, `"remote worker"`
- **No payload data**: Sensitive data never logged (sent to agent only)

---

## Recommendations

1. **Production**: `DD_TRACE_DEBUG=false`
2. **Profiling issues**: Enable temporarily for gem compatibility diagnostics
3. **AppSec issues**: Enable for security engine initialization details
4. **Remote Config issues**: Enable for client sync error details
5. **Performance**: Lazy evaluation means minimal overhead when debug disabled

---

## Code References

- Profiling: `/lib/datadog/profiling/component.rb`
- AppSec: `/lib/datadog/appsec/component.rb`
- Remote Config: `/lib/datadog/core/remote/client.rb`
- Telemetry: `/lib/datadog/core/telemetry/`
- Data Streams: `/lib/datadog/data_streams/`
- Core Logger: `/lib/datadog/core/logger.rb`
