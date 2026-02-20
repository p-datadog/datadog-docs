# Environment Variable Time Unit Conventions Across Datadog Tracers

**Analysis Date:** 2026-02-20
**Purpose:** Document naming conventions for time-related environment variables across Ruby, Java, Node.js, and Go Agent tracers

## Executive Summary

This document analyzes how time-related environment variables are named across Datadog's tracers. Key findings:

- **Ruby:** Never uses `_MS` suffix (0 occurrences), occasionally uses `_SECONDS` (3 times)
- **Java:** Rarely uses `_MILLIS` suffix (5 occurrences), rarely uses `_SECONDS` (2 times), mostly no suffix
- **Node.js:** Uses `_MS` suffix sparingly (2 occurrences, both unique to Node), uses `_SECONDS` for shared configs
- **Go Agent:** Never uses unit suffixes, relies on Go's `time.Duration` parsing

**Consistency:** Most time-related environment variables across all languages **do not include unit suffixes**, with units documented via property names, code comments, or documentation.

---

## Ruby Tracer (dd-trace-rb)

### All Time-Related Environment Variables

| Environment Variable | Unit | Default | Property Name | Notes |
|---------------------|------|---------|---------------|-------|
| `DD_AI_GUARD_TIMEOUT` | milliseconds | 10,000 | `timeout_ms` | AI Guard request timeout |
| `DD_API_SECURITY_SAMPLE_DELAY` | seconds | 30 | `sample_delay` | Delay before sampling (converted to int) |
| `DD_APPSEC_WAF_TIMEOUT` | microseconds | 5,000 | `waf_timeout` | WAF execution timeout (comment: `# us`) |
| `DD_PROFILING_UPLOAD_PERIOD` | seconds | 60 | `upload_period_seconds` | How often profiler reports data |
| `DD_PROFILING_UPLOAD_TIMEOUT` | seconds | 30.0 | `timeout_seconds` | Network timeout for profiling upload |
| `DD_REMOTE_CONFIG_BOOT_TIMEOUT_SECONDS` | seconds | 1.0 | `boot_timeout_seconds` | Remote config boot timeout |
| `DD_REMOTE_CONFIG_POLL_INTERVAL_SECONDS` | seconds | 5.0 | `poll_interval_seconds` | Remote config polling interval |
| `DD_TELEMETRY_HEARTBEAT_INTERVAL` | seconds | 60.0 | `heartbeat_interval_seconds` | Telemetry heartbeat interval |
| `DD_TELEMETRY_METRICS_AGGREGATION_INTERVAL` | seconds | 10.0 | `metrics_aggregation_interval_seconds` | Telemetry metrics aggregation interval |
| `DD_TRACE_AGENT_TIMEOUT_SECONDS` | seconds | 30 | `timeout_seconds` | Agent APM timeout (HTTP: 30, UDS: 1) |
| `OTEL_EXPORTER_OTLP_TIMEOUT` | milliseconds | 10,000 | `timeout_millis` | OTLP exporter timeout |
| `OTEL_EXPORTER_OTLP_METRICS_TIMEOUT` | milliseconds | 10,000 | `timeout_millis` | OTLP metrics-specific timeout |
| `OTEL_METRIC_EXPORT_INTERVAL` | milliseconds | 10,000 | `export_interval_millis` | Metrics export interval |
| `OTEL_METRIC_EXPORT_TIMEOUT` | milliseconds | 7,500 | `export_timeout_millis` | Metrics export timeout |

### Ruby Naming Patterns

- **Unit Suffixes:**
  - `_SECONDS` suffix: 3 occurrences (explicit)
  - `_MS` suffix: **0 occurrences**
  - `_MILLIS` suffix: **0 occurrences**
  - No suffix: 11 occurrences

- **Unit Distribution:**
  - Seconds: 9 variables
  - Milliseconds: 4 variables (OTEL only)
  - Microseconds: 1 variable (WAF)

- **Convention:** Units communicated through property names (`timeout_ms`, `upload_period_seconds`), code comments, or documentation

---

## Java Tracer (dd-trace-java)

### All Time-Related Environment Variables

| Environment Variable | Unit | Default | Config Property | Notes |
|---------------------|------|---------|-----------------|-------|
| `DD_AI_GUARD_TIMEOUT` | milliseconds | 10,000 | `ai_guard.timeout` | AI Guard request timeout |
| `DD_API_SECURITY_SAMPLE_DELAY` | seconds | 30 | `api-security.sample.delay` | API security sampling delay |
| `DD_APPSEC_REPORT_TIMEOUT_SEC` | seconds | - | `appsec.report.timeout` | AppSec report timeout |
| `DD_APPSEC_WAF_TIMEOUT` | microseconds | 10,000 | `appsec.waf.timeout` | WAF execution timeout |
| `DD_CIVISIBILITY_GIT_UPLOAD_TIMEOUT_MILLIS` | milliseconds | - | `civisibility.git.upload.timeout.millis` | Git upload timeout |
| `DD_CIVISIBILITY_GIT_COMMAND_TIMEOUT_MILLIS` | milliseconds | - | `civisibility.git.command.timeout.millis` | Git command timeout |
| `DD_CIVISIBILITY_BACKEND_API_TIMEOUT_MILLIS` | milliseconds | - | `civisibility.backend.api.timeout.millis` | Backend API timeout |
| `DD_CIVISIBILITY_SIGNAL_CLIENT_TIMEOUT_MILLIS` | milliseconds | - | `civisibility.signal.client.timeout.millis` | Signal client timeout |
| `DD_CRASHTRACKING_UPLOAD_TIMEOUT` | seconds | 10 | `crashtracking.upload.timeout` | Crash tracking upload timeout |
| `DD_DOGSTATSD_START_DELAY` | seconds | 0 | `dogstatsd.start-delay` | DogStatsD start delay |
| `DD_DYNAMIC_INSTRUMENTATION_CAPTURE_TIMEOUT` | milliseconds | 100 | `dynamic.instrumentation.capture.timeout` | **DI capture timeout** |
| `DD_DYNAMIC_INSTRUMENTATION_UPLOAD_TIMEOUT` | seconds | 30 | `dynamic.instrumentation.upload.timeout` | DI upload timeout |
| `DD_DYNAMIC_INSTRUMENTATION_UPLOAD_FLUSH_INTERVAL` | milliseconds | 0 | `dynamic.instrumentation.upload.flush.interval` | DI flush interval (0 = dynamic) |
| `DD_DYNAMIC_INSTRUMENTATION_UPLOAD_INTERVAL_SECONDS` | seconds | - | `dynamic.instrumentation.upload.interval.seconds` | DI upload interval |
| `DD_DYNAMIC_INSTRUMENTATION_POLL_INTERVAL` | seconds | 1 | `dynamic.instrumentation.poll.interval` | DI polling interval |
| `DD_DYNAMIC_INSTRUMENTATION_DIAGNOSTICS_INTERVAL` | seconds | 3,600 | `dynamic.instrumentation.diagnostics.interval` | DI diagnostics interval |
| `DD_DEBUGGER_EXCEPTION_CAPTURE_INTERVAL_SECONDS` | seconds | 3,600 | `debugger.exception.capture.interval.seconds` | Exception capture interval |
| `DD_JMX_FETCH_START_DELAY` | seconds | 500 | `jmxfetch.start-delay` | JMX fetch start delay |
| `DD_JMX_FETCH_CHECK_PERIOD` | seconds | - | `jmxfetch.check-period` | JMX fetch check period |
| `DD_JMX_FETCH_REFRESH_BEANS_PERIOD` | seconds | - | `jmxfetch.refresh-beans-period` | JMX refresh beans period |
| `DD_PROFILING_START_DELAY` | seconds | 10 | `profiling.start-delay` | Profiling start delay |
| `DD_PROFILING_UPLOAD_PERIOD` | seconds | 60 | `profiling.upload.period` | Profiling upload period |
| `DD_PROFILING_UPLOAD_TIMEOUT` | seconds | 30 | `profiling.upload.timeout` | Profiling upload timeout |
| `DD_PROFILING_DDPROF_CPU_INTERVAL` | milliseconds | 10 | `profiling.ddprof.cpu.interval.ms` | CPU profiling interval |
| `DD_PROFILING_DDPROF_WALL_INTERVAL` | milliseconds | 50 | `profiling.ddprof.wall.interval.ms` | Wall profiling interval |
| `DD_REMOTE_CONFIG_POLL_INTERVAL_SECONDS` | seconds | 5 | `remote-config.poll.interval.seconds` | Remote config poll interval |
| `DD_STATSD_CLIENT_SOCKET_TIMEOUT` | milliseconds | - | `statsd.client.socket.timeout` | StatsD socket timeout |
| `DD_TELEMETRY_HEARTBEAT_INTERVAL` | seconds | 60 | `telemetry.heartbeat.interval` | Telemetry heartbeat interval |
| `DD_TELEMETRY_METRICS_INTERVAL` | seconds | 10 | `telemetry.metrics.interval` | Telemetry metrics interval |
| `DD_TRACE_AGENT_TIMEOUT` | seconds | 10 | `trace.agent.timeout` | Trace agent timeout |
| `DD_TRACE_CLOCK_SYNC_PERIOD` | seconds | 60 | `trace.clock.sync.period` | Clock sync period |
| `DD_TRACE_FLUSH_INTERVAL` | seconds | 1 | `trace.flush.interval` | Trace flush interval |

### Java Naming Patterns

- **Unit Suffixes:**
  - `_MILLIS` suffix: 5 occurrences (CiVisibility only)
  - `_SECONDS` suffix: 2 occurrences
  - No suffix: 24+ occurrences

- **Convention:** Most variables have **no unit suffix**, even when value is in milliseconds. Units inferred from property names or defaults.

---

## Node.js Tracer (dd-trace-js)

### All Time-Related Environment Variables

| Environment Variable | Unit | Default | Config Property | Notes |
|---------------------|------|---------|-----------------|-------|
| `DD_AI_GUARD_TIMEOUT` | milliseconds | 10,000 | `experimental.aiguard.timeout` | AI Guard request timeout |
| `DD_API_SECURITY_SAMPLE_DELAY` | seconds | 30 | `appsec.apiSecurity.sampleDelay` | API security sampling delay |
| `DD_APPSEC_WAF_TIMEOUT` | microseconds | 10,000 | `appsec.wafTimeout` | WAF execution timeout |
| `DD_DYNAMIC_INSTRUMENTATION_CAPTURE_TIMEOUT_MS` | milliseconds | 15 | `dynamicInstrumentation.captureTimeoutMs` | **DI capture timeout (MS suffix!)** |
| `DD_DYNAMIC_INSTRUMENTATION_UPLOAD_INTERVAL_SECONDS` | seconds | 1 | `dynamicInstrumentation.uploadIntervalSeconds` | DI upload interval |
| `DD_EXPERIMENTAL_FLAGGING_PROVIDER_INITIALIZATION_TIMEOUT_MS` | milliseconds | 30,000 | `experimental.flaggingProvider.initializationTimeoutMs` | Feature flag init timeout |
| `DD_HEAP_SNAPSHOT_INTERVAL` | seconds | 3,600 | `heapSnapshot.interval` | Heap snapshot interval |
| `DD_REMOTE_CONFIG_POLL_INTERVAL_SECONDS` | seconds | 5 | `remoteConfig.pollInterval` | Remote config poll interval |
| `DD_TELEMETRY_HEARTBEAT_INTERVAL` | seconds | 60 | `telemetry.heartbeatInterval` | Telemetry heartbeat interval |
| `DD_TRACE_FLUSH_INTERVAL` | milliseconds | 2,000 | `flushInterval` | Trace flush interval |

### Node.js Naming Patterns

- **Unit Suffixes:**
  - `_MS` suffix: **2 occurrences** (both unique to Node.js)
    - `DD_DYNAMIC_INSTRUMENTATION_CAPTURE_TIMEOUT_MS`
    - `DD_EXPERIMENTAL_FLAGGING_PROVIDER_INITIALIZATION_TIMEOUT_MS`
  - `_SECONDS` suffix: 2 occurrences (shared configs)
  - No suffix: 6 occurrences

- **Convention:** Uses `_MS` suffix for Node-specific configs, no suffix for shared cross-language configs

---

## Go Agent (datadog-agent)

### Circuit Breaker Configuration

The Go agent uses a different approach - no individual environment variables for time values, but rather nested configuration:

| Configuration Key | Unit | Default | Notes |
|------------------|------|---------|-------|
| `dynamic_instrumentation.circuit_breaker.interval` | duration | 1s | Check interval (Go duration string) |
| `dynamic_instrumentation.circuit_breaker.per_probe_cpu_limit` | CPUs/s | 0.1 | CPU limit per probe per core |
| `dynamic_instrumentation.circuit_breaker.all_probes_cpu_limit` | CPUs/s | 0.5 | Total CPU limit per core |
| `dynamic_instrumentation.circuit_breaker.interrupt_overhead` | duration | 5µs | Cost per probe hit |

**Go Convention:** Uses `time.Duration` parsing which accepts unit suffixes in values (e.g., `"1s"`, `"5µs"`), not in variable names.

---

## Cross-Language Comparison: Same Settings, Different Names

| Setting | Ruby | Java | Node.js | Notes |
|---------|------|------|---------|-------|
| **AI Guard Timeout** | `DD_AI_GUARD_TIMEOUT` (ms) | `DD_AI_GUARD_TIMEOUT` (ms) | `DD_AI_GUARD_TIMEOUT` (ms) | ✅ Perfect consistency |
| **API Security Sample Delay** | `DD_API_SECURITY_SAMPLE_DELAY` (s) | `DD_API_SECURITY_SAMPLE_DELAY` (s) | `DD_API_SECURITY_SAMPLE_DELAY` (s) | ✅ Perfect consistency |
| **AppSec WAF Timeout** | `DD_APPSEC_WAF_TIMEOUT` (µs) | `DD_APPSEC_WAF_TIMEOUT` (µs) | `DD_APPSEC_WAF_TIMEOUT` (µs) | ✅ Perfect consistency |
| **DI Capture Timeout** | ❌ N/A | `DD_DYNAMIC_INSTRUMENTATION_CAPTURE_TIMEOUT` (ms) | `DD_DYNAMIC_INSTRUMENTATION_CAPTURE_TIMEOUT_MS` (ms) | ❌ Node adds `_MS` suffix |
| **Profiling Upload Period** | `DD_PROFILING_UPLOAD_PERIOD` (s) | `DD_PROFILING_UPLOAD_PERIOD` (s) | ❌ N/A | ✅ Consistent where exists |
| **Profiling Upload Timeout** | `DD_PROFILING_UPLOAD_TIMEOUT` (s) | `DD_PROFILING_UPLOAD_TIMEOUT` (s) | ❌ N/A | ✅ Consistent where exists |
| **Remote Config Poll Interval** | `DD_REMOTE_CONFIG_POLL_INTERVAL_SECONDS` | `DD_REMOTE_CONFIG_POLL_INTERVAL_SECONDS` | `DD_REMOTE_CONFIG_POLL_INTERVAL_SECONDS` | ✅ Perfect consistency with explicit `_SECONDS` |
| **Telemetry Heartbeat Interval** | `DD_TELEMETRY_HEARTBEAT_INTERVAL` (s) | `DD_TELEMETRY_HEARTBEAT_INTERVAL` (s) | `DD_TELEMETRY_HEARTBEAT_INTERVAL` (s) | ✅ Perfect consistency |
| **Trace Agent Timeout** | `DD_TRACE_AGENT_TIMEOUT_SECONDS` | `DD_TRACE_AGENT_TIMEOUT` (s) | ❌ N/A | ⚠️ Ruby adds `_SECONDS`, Java doesn't |
| **Trace Flush Interval** | ❌ N/A | `DD_TRACE_FLUSH_INTERVAL` (s) | `DD_TRACE_FLUSH_INTERVAL` (ms) | ⚠️ **Different units!** Java=seconds, Node=milliseconds |

---

## Key Findings

### 1. Predominant Pattern: No Unit Suffix

**Across all languages, the vast majority of time-related environment variables DO NOT include unit suffixes:**

- Ruby: 11 of 14 (79%)
- Java: 24+ of 31 (77%)
- Node.js: 6 of 10 (60%)

### 2. Explicit Unit Suffixes Are Rare

When unit suffixes appear:
- `_SECONDS` suffix: Used for disambiguation or emphasis (Ruby: 3, Java: 2, Node: 2)
- `_MILLIS` suffix: Java only, limited to CiVisibility (5 occurrences)
- `_MS` suffix: Node.js only, for Node-specific features (2 occurrences)

### 3. Cross-Language Consistency is Generally Good

Shared configurations typically use **identical names**:
- `DD_AI_GUARD_TIMEOUT`
- `DD_API_SECURITY_SAMPLE_DELAY`
- `DD_APPSEC_WAF_TIMEOUT`
- `DD_TELEMETRY_HEARTBEAT_INTERVAL`
- `DD_REMOTE_CONFIG_POLL_INTERVAL_SECONDS` (with explicit `_SECONDS`)

### 4. Node.js is the Outlier

Node.js uniquely adds `_MS` suffix to its Dynamic Instrumentation configs:
- `DD_DYNAMIC_INSTRUMENTATION_CAPTURE_TIMEOUT_MS` (Java: `DD_DYNAMIC_INSTRUMENTATION_CAPTURE_TIMEOUT`)
- This breaks consistency with Java's naming pattern

### 5. Units Are Communicated Through Other Means

Instead of env var suffixes, units are documented via:
- Property/option names (e.g., `timeout_ms`, `upload_period_seconds`)
- Code comments (e.g., `# us`, `// milliseconds`)
- Default value constants (e.g., `DEFAULT_TIMEOUT = 100 // milliseconds`)
- Documentation

---

## Recommendations for New Environment Variables

### Rule 1: Follow Language-Specific Patterns

**Ruby:**
- ✅ DO: Use no suffix for most cases
- ✅ DO: Use `_SECONDS` suffix only when needed for clarity
- ❌ DON'T: Use `_MS` or `_MILLIS` suffix

**Java:**
- ✅ DO: Use no suffix for most cases
- ✅ DO: Use `_MILLIS` suffix for Java-specific low-level features (rare)
- ❌ DON'T: Use `_MS` suffix

**Node.js:**
- ✅ DO: Use no suffix for shared cross-language configs
- ⚠️ CONSIDER: `_MS` suffix for Node-specific features (establishes precedent)

### Rule 2: Prioritize Cross-Language Consistency

When adding a config that exists in other tracers:
- Use the **same name** as other languages
- Use the **same units** as other languages
- Document any unavoidable differences

### Rule 3: Document Units Explicitly

Always document units through:
1. Property name (preferred: `timeout_ms`, `period_seconds`)
2. Code comments
3. Documentation/changelog entries
4. Default value constant comments

### Rule 4: New Ruby Configs Should NOT Use `_MS`

Based on analysis:
- Ruby has **zero** existing `_MS` suffixes
- Ruby follows Java's pattern (no `_MS`)
- Adding `_MS` would break Ruby's established convention

---

## Analysis Methodology

This document was created by:
1. Searching all time-related configuration in Ruby tracer (`dd-trace-rb`)
2. Searching all time-related configuration in Java tracer (`dd-trace-java`)
3. Searching all time-related configuration in Node.js tracer (`dd-trace-js`)
4. Examining Go agent's circuit breaker implementation
5. Cross-referencing shared configurations across languages
6. Analyzing naming patterns and unit conventions

---

**Document Version:** 1.0
**Last Updated:** 2026-02-20
