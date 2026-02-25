# DTR (dd-trace-rb) Logging Configuration Guide

## Overview

This guide explains how to configure logging for Datadog's Ruby tracer (dtr/dd-trace-rb) to get only tracer logs or only Dynamic Instrumentation (DI) logs.

## What DD_TRACE_DEBUG Does

When `DD_TRACE_DEBUG` environment variable is set to `true` or `1`, it enables:

### 1. Sets Logger Level to DEBUG
```ruby
logger.level = settings.diagnostics.debug ? ::Logger::DEBUG : settings.logger.level
```
This enables **all** `logger.debug` calls throughout the codebase (129+ locations).

### 2. Adds Caller Stack Traces to ALL Log Messages
When debug mode is on, **every log message** (not just debug level) includes the caller location:
```ruby
# From lib/datadog/core/logger.rb:24
if debug? || severity >= ::Logger::ERROR
  c = caller
  where = "(#{c[1]}) " if c.length > 1
end
```

**Example output**:
```
[datadog] (lib/datadog/tracing/tracer.rb:567) Writing 3 spans (enabled: true)
```

### 3. Logs Complete Span Details for Every Trace

When a trace is written, it logs **all spans with full details** via `pretty_inspect`:

```ruby
# lib/datadog/tracing/tracer.rb:566-567
if Datadog.configuration.diagnostics.debug
  logger.debug { "Writing #{trace.length} spans (enabled: #{@enabled})\n#{trace.spans.pretty_inspect}" }
end
```

**Output per span** (~20-30 lines each):
```
Writing 3 spans (enabled: true)
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
   span.kind => server,
   env => production
 ]
 Metrics: [
   _dd.top_level => 1.0,
   _sampling_priority_v1 => 1.0
 ]
 Metastruct: []
,
 ... (repeats for every span in trace)
]
```

### 4. Logs Operational Debug Messages

Including:
- Configuration changes: `"Reconfigured tracer option..."`
- Transport errors: `"send_request failed: #{response}"`
- HTTP failures: `"Internal error during HTTP request"`
- Integration errors: `"Error while tracing Rake invoke"`
- Distributed tracing: `"Cannot inject distributed trace data: digest is nil"`
- Pipeline errors: `"trace dropped entirely due to error"`
- Writer issues: `"Writer failed to start"`

## DD_TRACE_DEBUG and Dynamic Instrumentation

`DD_TRACE_DEBUG` also affects DI in two ways:

### 1. Enables Standard Debug Logs (~26 statements)

All `logger.debug` calls show:
```
di: activating code tracking
di: loading contrib/railtie
di: loading contrib/active_record
di: received log probe at users_controller.rb:45 (probe-123) via RC
di: unhandled exception in script_compiled trace point: RuntimeError: ...
di: error in probe notifier worker: StandardError: ... (at lib/...)
di: dropping status for log probe at users_controller.rb:45 (probe-123): INSTALLED because queue is full
di: failed to send snapshot: Errno::ECONNREFUSED: ... (at lib/...)
di: error processing probe configuration: ArgumentError: ...
```

### 2. Enables Extra Verbose "Trace" Logs (~9 statements)

`DD_TRACE_DEBUG` also sets `settings.dynamic_instrumentation.internal.trace_logging = true`, which enables all `logger.trace` calls:

```
di: starting probe notifier: pid 12345
di: installed log probe at users_controller.rb:45 (probe-123)
di: could not install probe... adding it to pending list
di: executed log probe at users_controller.rb:45 (probe-123)  # ← fires EVERY time probe hits
di: queueing status for log probe at users_controller.rb:45 (probe-123): INSTALLED
di: queueing snapshot event
di: checking snapshot queue - 1 entries
di: sending 1 snapshot event(s) to agent
di: stopping probe notifier: pid 12345
```

**Note**: The most impactful is `"di: executed #{probe.type} probe at..."` which logs **every time a probe fires**, potentially thousands of times per minute for hot paths.

**Important**: The actual DI snapshot data (with all the captured variables, stack traces, etc.) is **never logged** - it's only sent directly to the agent via the transport layer.

## Configuration Options

| Setting | Controls | Default | Env Var |
|---------|----------|---------|---------|
| `diagnostics.debug` | Logger level + Tracer span output | `false` | `DD_TRACE_DEBUG` |
| `dynamic_instrumentation.internal.trace_logging` | DI verbose trace logs | `false` | `DD_TRACE_DEBUG` |
| `logger.level` | Base logger level | `INFO` | - |

**Problem**: `DD_TRACE_DEBUG` env var sets **both** `diagnostics.debug` and `trace_logging` to `true`.

**Solution**: Configure programmatically to override the env var behavior.

---

## Scenario 1: Only Tracer Debug Logs (No DI Logs)

### Method A: Programmatic Configuration
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true  # Enables tracer span output + sets logger to DEBUG
  c.dynamic_instrumentation.internal.trace_logging = false  # Disables DI trace logs
end
```

**What you get**:
```
[datadog] Writing 3 spans (enabled: true)
[
 Name: rack.request
 Span ID: 1234567890123456789
 ...
]
[datadog] Reconfigured tracer option...
[datadog] Internal error during HTTP request...
```

**What you DON'T get**:
- DI trace logs (`di: executed probe...`, `di: queueing snapshot...`)

**What you WILL still get**:
- DI debug logs (`di: error processing probe...`) because the logger level is DEBUG

### Method B: Filter Out DI Logs
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true

  # Custom logger that filters out DI messages
  original_logger = c.logger.instance || Datadog::Core::Logger.new($stderr)
  filtered_logger = Object.new.tap do |logger|
    define_singleton_method(:add) do |severity, message = nil, progname = nil, &block|
      msg = message || (block && block.call) || progname
      return if msg.to_s.include?('di:')  # Filter out DI logs
      original_logger.add(severity, message, progname, &block)
    end

    define_singleton_method(:debug?) { original_logger.debug? }
    # ... delegate other methods as needed
  end

  c.logger.instance = filtered_logger
end
```

---

## Scenario 2: Only DI Debug Logs (No Tracer Span Output)

### Method A: Disable Tracer Span Output
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = false  # Disables tracer span pretty_inspect output
  c.logger.level = ::Logger::DEBUG  # Manually set logger to DEBUG
  c.dynamic_instrumentation.internal.trace_logging = true  # Enables DI trace logs
end
```

**What you get**:
```
[datadog] di: starting probe notifier: pid 12345
[datadog] di: received log probe at users_controller.rb:45 (probe-123) via RC
[datadog] di: installed log probe at users_controller.rb:45 (probe-123)
[datadog] di: executed log probe at users_controller.rb:45 (probe-123)
[datadog] di: queueing snapshot event
[datadog] di: error processing probe configuration: ...
```

**What you DON'T get**:
- Tracer span output (`Writing X spans... [Name: ..., Span ID: ...]`)

**What you WILL still get**:
- Regular tracer debug logs (errors, warnings, operational messages) because logger level is DEBUG

### Method B: Filter Out Tracer Messages
```ruby
Datadog.configure do |c|
  c.logger.level = ::Logger::DEBUG
  c.dynamic_instrumentation.internal.trace_logging = true

  # Custom logger that filters out tracer messages
  original_logger = c.logger.instance || Datadog::Core::Logger.new($stderr)
  filtered_logger = Object.new.tap do |logger|
    define_singleton_method(:add) do |severity, message = nil, progname = nil, &block|
      msg = message || (block && block.call) || progname
      # Filter out tracer-specific patterns
      return if msg.to_s.match?(/Writing \d+ spans|Name:|Span ID:|Trace ID:/)
      original_logger.add(severity, message, progname, &block)
    end

    define_singleton_method(:debug?) { original_logger.debug? }
    # ... delegate other methods as needed
  end

  c.logger.instance = filtered_logger
end
```

---

## Scenario 3: All Debug Logs (Default DD_TRACE_DEBUG Behavior)

### Environment Variable
```bash
export DD_TRACE_DEBUG=true
```

### Programmatic
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = true
  c.dynamic_instrumentation.internal.trace_logging = true
end
```

---

## Scenario 4: No Debug Logs at All

### Environment Variable
```bash
# Don't set DD_TRACE_DEBUG
unset DD_TRACE_DEBUG
```

### Programmatic
```ruby
Datadog.configure do |c|
  c.diagnostics.debug = false
  c.logger.level = ::Logger::INFO  # or higher
  c.dynamic_instrumentation.internal.trace_logging = false
end
```

---

## Summary Table

| Goal | `diagnostics.debug` | `logger.level` | `DI trace_logging` | Notes |
|------|-------------------|----------------|-------------------|-------|
| **Only tracer spans** | `true` | (auto-set to DEBUG) | `false` | Disables DI trace logs but keeps DI debug logs |
| **Only DI logs** | `false` | `::Logger::DEBUG` | `true` | Disables tracer spans but keeps tracer debug logs |
| **All debug logs** | `true` | (auto-set to DEBUG) | `true` | Default when `DD_TRACE_DEBUG=true` |
| **No debug logs** | `false` | `::Logger::INFO` | `false` | Default when DD_TRACE_DEBUG not set |

---

## Key Insights

1. **Shared Logger**: Tracer and DI share the same logger and logger level. There's no built-in way to completely isolate tracer vs DI logs.

2. **DD_TRACE_DEBUG is Global**: Setting `DD_TRACE_DEBUG` environment variable enables verbose logging for both tracer and DI.

3. **Programmatic Override**: The best approach is to:
   - Disable the verbose outputs (`diagnostics.debug` for tracer spans, `trace_logging` for DI verbose logs)
   - Set logger level manually if needed
   - Filter remaining debug messages with a custom logger if needed

4. **DI Snapshot Data**: The actual DI snapshot data (captured variables, stack traces, etc.) is **never logged to the logger** - it's only sent to the agent via the transport layer.

5. **Verbosity Levels**:
   - **Tracer**: Logs full span details (~50-150 lines per trace)
   - **DI**: Logs operational events (~1-10 lines per probe hit, depending on activity)

---

## Code References

- Tracer span logging: `/real.home/claude-5/dtr/lib/datadog/tracing/tracer.rb:566-567`
- DI trace logging configuration: `/real.home/claude-5/dtr/lib/datadog/di/configuration/settings.rb:216-226`
- DI Logger wrapper: `/real.home/claude-5/dtr/lib/datadog/di/logger.rb`
- Core logger with stack traces: `/real.home/claude-5/dtr/lib/datadog/core/logger.rb:24-27`
- Core diagnostics debug setting: `/real.home/claude-5/dtr/lib/datadog/core/configuration/settings.rb:115-143`
