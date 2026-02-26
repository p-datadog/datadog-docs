# Senior Engineer Architecture Review

## Primary Goal: Does Core have zero dependency on products?

---

## ✅ **PASS: Core has no requires of product files**

**Before refactor:**
```ruby
# lib/datadog/core/configuration/components.rb (IN CORE)
require_relative '../../tracing/component'      # Product!
require_relative '../../profiling/component'    # Product!
require_relative '../../appsec/component'       # Product!
# ... all products
```

**After refactor:**
```ruby
# lib/datadog/components.rb (TOP LEVEL)
require_relative 'tracing/component'
require_relative 'profiling/component'
# ... all products
```
✓ Core no longer requires component files

**Settings extends:**
```ruby
# lib/datadog/core/configuration/settings.rb (IN CORE)
require_relative '../../tracing/configuration/settings'        # Product!
require_relative '../../opentelemetry/configuration/settings'  # Product!
extend Datadog::Tracing::Configuration::Settings
extend Datadog::OpenTelemetry::Configuration::Settings
```

**After refactor:**
```ruby
# lib/datadog/configuration_loader.rb (TOP LEVEL)
require_relative 'tracing/configuration/settings'
require_relative 'opentelemetry/configuration/settings'
Datadog::Settings.extend(Tracing::Configuration::Settings)
```
✓ Core no longer requires or extends product settings

---

## ⚠️ **CONCERN: What about profiling Ext constants?**

**Current state (line 12 of core/configuration/settings.rb):**
```ruby
require_relative '../../profiling/ext'  # Product require IN CORE!
```

Then used in profiling settings:
```ruby
o.env Profiling::Ext::ENV_ENABLED
```

**After extracting profiling settings to module:**
```ruby
# lib/datadog/profiling/configuration/settings.rb (PRODUCT)
require_relative '../ext'

module Datadog::Profiling::Configuration::Settings
  settings :profiling do
    option :enabled do |o|
      o.env Profiling::Ext::ENV_ENABLED
    end
  end
end
```

✓ Profiling Ext require moves to product module
✓ Core no longer requires profiling/ext

**Verify during implementation:** Ensure no other product Ext constants are used in CoreSettings

---

## ✅ **PASS: Core has no knowledge of specific products**

**CoreSettings module contains:**
- `service`, `env`, `tags` - infrastructure
- `agent.*` - infrastructure
- `runtime_metrics` - infrastructure (supports all products)
- `health_metrics` - infrastructure (supports all products)
- `telemetry` - infrastructure (supports all products)
- `remote` - infrastructure (supports all products)
- `logger`, `diagnostics` - infrastructure

**CoreSettings does NOT contain:**
- tracing options (in Tracing::Configuration::Settings)
- profiling options (extracting to Profiling::Configuration::Settings)
- appsec options (in AppSec::Configuration::Settings)
- ai_guard options (in AIGuard::Configuration::Settings)
- etc.

✓ Core defines only infrastructure options

---

## ✅ **PASS: Clean layer separation**

**After refactor:**

```
┌─────────────────────────────────────────────┐
│ Top Level (lib/datadog/)                   │
│ - Settings (storage)                        │
│ - Components (product instantiation)        │
│ - ComponentLifecycle (component management) │
│ - configuration_loader (orchestration)      │
│ - Datadog module (public API)              │
│                                             │
│ Requires: Core + All Products              │
└─────────────────────────────────────────────┘
                    ↓ uses
┌─────────────────────────────────────────────┐
│ Core (lib/datadog/core/)                   │
│ - Configuration::Base (DSL)                 │
│ - Configuration::CoreSettings (core opts)   │
│ - Configuration (Pin utilities)             │
│                                             │
│ Requires: Nothing outside Core             │
└─────────────────────────────────────────────┘
```

✓ Dependency flows one direction (top → core), never reverse

---

## ✅ **PASS: State vs Infrastructure separation**

| Concept | Before | After |
|---------|--------|-------|
| Settings DSL | Core | Core (Base) ✓ |
| Settings Storage | Core | Top-level ✓ |
| Settings Values | Core | Top-level ✓ |
| Components Class | Core | Top-level ✓ |
| Component Lifecycle | Core | Top-level ✓ |
| Pin Utilities | Core | Core ✓ |

✓ Core = tools, Top-level = state

---

## 🔍 **Areas to verify during implementation:**

### 1. CoreSettings doesn't reference product classes
```ruby
# BAD (if this exists):
option :default_tracer, type: Datadog::Tracing::Tracer  # References product class!

# GOOD:
option :service, type: :string  # Primitive type only
```

### 2. No hidden transitive dependencies
- Check that CoreSettings doesn't use any constants from product Ext modules
- Check that Core utilities don't call product-specific code

### 3. "Core only" mode still works
If someone does:
```ruby
require 'datadog/core'  # Without requiring products
```
They should get:
- Core infrastructure ✓
- No products loaded ✓
- No errors ✓

---

## 📊 **Completeness Check**

Does the proposal cover everything needed to remove dependencies?

✅ Settings storage moved to top level
✅ Components moved to top level
✅ Component lifecycle moved to top level
✅ Product settings extends moved to orchestrator
✅ Product component requires moved to top level
✅ Core keeps only infrastructure
✅ All references updated

---

## 🎯 **Final Verdict**

**Architecture: SOUND ✓**

The proposal successfully removes Core's dependency on products:
- Core requires zero product files
- Core has zero knowledge of specific products
- Core provides only infrastructure (DSL, utilities, core options)
- All state and orchestration moved to top level
- Clean layer separation maintained

**Recommendations:**
1. During implementation, verify CoreSettings has no product Ext references
2. During implementation, verify profiling settings extraction is clean
3. Test "core only" mode after refactor
4. Ensure no transitive dependencies snuck in

**This proposal achieves the architectural goal.**
