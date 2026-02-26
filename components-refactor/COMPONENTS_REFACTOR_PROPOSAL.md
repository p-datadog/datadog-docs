# Proposal: Lift Components and Settings to Top Level

## Problem
Core has dependencies on products in two areas:

1. **Components**: `Datadog::Core::Configuration::Components` requires all product component files (tracing, profiling, appsec, etc.)
2. **Settings Storage**: `Datadog::Core::Configuration::Settings` class stores runtime configuration values and gets extended by products

This prevents Core from being truly independent.

## Solution
Move Components, component lifecycle management, and Settings storage to top level. Core retains only infrastructure (DSL, utilities).

## Changes

### 1. Move Settings Storage to Top Level
- `lib/datadog/core/configuration/settings.rb` → `lib/datadog/settings.rb`
- Namespace: `Datadog::Core::Configuration::Settings` → `Datadog::Settings`
- Core keeps DSL infrastructure:
  - `Core::Configuration::Base` (option/settings DSL)
  - `Core::Configuration::Option`
  - `Core::Configuration::OptionDefinition`
- Core's options extracted to extension module:
  - Create `lib/datadog/core/configuration/core_settings.rb`
  - Module: `Datadog::Core::Configuration::CoreSettings`
  - Contains options like `service`, `env`, `tags`, `agent.*`
- Top-level Settings class includes Core's Base and gets extended by all settings modules
- Top-level orchestrator extends Settings with CoreSettings + ProductSettings

### 2. Move Components
- `lib/datadog/core/configuration/components.rb` → `lib/datadog/components.rb`
- Namespace: `Datadog::Core::Configuration::Components` → `Datadog::Components`

### 3. Create new top-level module for component lifecycle
- Create `lib/datadog/component_lifecycle.rb`
- New module: `Datadog::ComponentLifecycle`
- Move these from `Datadog::Core::Configuration`:
  - `COMPONENTS_WRITE_LOCK`
  - `COMPONENTS_READ_LOCK`
  - `configure` (component rebuild/replace logic only)
  - `health_metrics`
  - `logger`
  - `config_init_logger`
  - `shutdown!`
  - `components` (protected)
  - `components?` (private)
  - `reset!` (private)
  - `safely_synchronize` (private)
  - `build_components` (private)
  - `replace_components!` (private)
  - `logger_without_components` (private)
  - `handle_interrupt_shutdown!` (private)

### 4. Core::Configuration becomes minimal utilities
`Datadog::Core::Configuration` retains only:
- `configure_onto` - per-object Pin configuration
- `configuration_for` - retrieves per-object Pin configuration
- `logger_without_configuration` - bootstrap logger from env vars

Core::Configuration no longer:
- Provides `configuration` method (moved to top-level Datadog)
- Stores the Settings class (moved to top level)
- Manages components (moved to ComponentLifecycle)
- Extends product settings (happens at top level)

### 5. Create top-level configuration orchestrator
- Create `lib/datadog/configuration_loader.rb`
- Requires Core's CoreSettings module
- Requires all product settings modules (Tracing, OpenTelemetry, etc.)
- Extends `Datadog::Settings` with all settings modules:
  ```ruby
  Datadog::Settings.extend(Core::Configuration::CoreSettings)
  Datadog::Settings.extend(Tracing::Configuration::Settings)
  Datadog::Settings.extend(OpenTelemetry::Configuration::Settings)
  ```

### 6. Update top-level Datadog module
Datadog defines directly:
- `configuration` - creates and returns `Datadog::Settings` instance
- `configuration?` - checks if configuration exists

Datadog extends:
- `Core::Configuration` (Pin utilities: configure_onto, configuration_for, logger_without_configuration)
- `ComponentLifecycle` (component orchestration: configure, shutdown!, build_components, etc.)

### 7. Update all references
- Files referencing `Datadog::Core::Configuration::Components` → `Datadog::Components`
- Files referencing `Datadog::Core::Configuration::Settings` → `Datadog::Settings`
- Files in `lib/`, `spec/`, and `sig/` directories

## Open Questions

### Q1: Core::Configuration.configuration method - RESOLVED
**Decision: Move to top-level Datadog module directly**

Rationale:
- Settings class lives at top level, so method that creates it should too
- Follows ownership principle: `Datadog::Settings` class → `Datadog.configuration` method
- Core::Configuration becomes pure utilities (Pin operations, bootstrap logger)
- Matches precedent from other libraries (Rails.configuration, Sidekiq.configure)

Implementation:
```ruby
# lib/datadog.rb
module Datadog
  class << self
    def configuration
      @configuration ||= Settings.new
    end

    def configuration?
      (defined?(@configuration) && @configuration) != nil
    end
  end

  extend Core::Configuration    # Pin utilities only
  extend ComponentLifecycle      # Component management
end
```

Core::Configuration retains only:
- `configure_onto` / `configuration_for` - Pin utilities
- `logger_without_configuration` - Bootstrap logger

### Q2: Interaction between Core::Configuration and ComponentLifecycle - RESOLVED
**Decision: No coordination needed - orthogonal concerns**

Rationale:
- Core::Configuration provides Pin utilities only (configure_onto, configuration_for, logger_without_configuration)
  - Standalone utilities, no state management
  - No interaction with components or configuration instance
- ComponentLifecycle manages component state (@components)
  - Calls `self.configuration` (Datadog's method) when needed
  - Doesn't touch Pin utilities
- Separate state management:
  - `@configuration` owned by Datadog directly
  - `@components` owned by ComponentLifecycle (extended into Datadog)
  - Pin configuration stored on target objects (via Pin)
- No interaction between the two modules - they just add independent methods to Datadog

Example:
```ruby
# ComponentLifecycle#configure calls Datadog.configuration
Datadog.configure { |c| c.service = 'foo' }
  → ComponentLifecycle#configure → Datadog.configuration → rebuilds components

# Core::Configuration#configure_onto is independent
Datadog.configure_onto(redis, service: 'cache')
  → Core::Configuration#configure_onto → Pin.set_on(redis) → no component interaction
```

### Q3: Profiling settings currently inline in Core - RESOLVED
**Decision: Extract to module like other products (verify during implementation)**

Context:
- Lines 238+ in `core/configuration/settings.rb` define profiling settings inline
- This is inconsistent with other products (Tracing, OpenTelemetry) which use extension modules

Recommendation:
- Extract profiling settings to `Profiling::Configuration::Settings` module
- Have orchestrator extend Settings with this module (consistent with other products)
- Treat profiling the same as tracing/appsec/etc - as a product, not core

Note:
- This is not critical to the architectural design
- Should be confirmed during implementation that extraction doesn't cause issues
- If problems arise, can keep inline as fallback

## Result
- Core provides only infrastructure (DSL, Pin utilities)
- Core has no storage classes (Settings, Components)
- Top-level owns all state (Settings + Components)
- Settings and Components orchestrated at top level where they can require all products
- No circular dependencies
