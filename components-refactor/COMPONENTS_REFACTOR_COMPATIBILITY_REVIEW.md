# Backward Compatibility Review - External Code Impact Analysis

## Methodology
Analyzing likelihood of external usage for each moved item:
- **PUBLIC_API tagged** = Very High Risk
- **Public methods on Datadog module** = High Risk
- **Public methods on internal classes** = Medium Risk
- **Protected/Private methods** = Low Risk
- **Internal constants** = Low-Medium Risk

---

## 1. Class Relocations

### 🔴 **HIGH RISK: `Datadog::Core::Configuration::Settings` → `Datadog::Settings`**

**Likelihood of external usage: HIGH**

**Potential external usage:**
```ruby
# Customer code checking class
if config.is_a?(Datadog::Core::Configuration::Settings)
  # ...
end

# Extension libraries instantiating directly
settings = Datadog::Core::Configuration::Settings.new

# Type checking in gems
def configure_datadog(settings: Datadog::Core::Configuration::Settings)
  # ...
end

# Metaprogramming/reflection
Datadog::Core::Configuration::Settings.instance_methods
```

**Risk assessment:**
- Class is not @public_api tagged
- BUT: It's the return type of `Datadog.configuration` (public API)
- Likely used in type checks, metaprogramming
- Medium-high probability of breakage

**Compatibility shim:**
```ruby
# lib/datadog/core/configuration/settings.rb
module Datadog
  module Core
    module Configuration
      # Compatibility shim for old class name
      Settings = ::Datadog::Settings

      # If Settings was a class before, might need:
      # class Settings < ::Datadog::Settings; end
      # to handle inheritance checks
    end
  end
end
```

---

### 🟡 **MEDIUM RISK: `Datadog::Core::Configuration::Components` → `Datadog::Components`**

**Likelihood of external usage: MEDIUM-LOW**

**Potential external usage:**
```ruby
# Test helpers in customer test suites
components = Datadog::Core::Configuration::Components.new(settings)

# Extension gems reaching into internals
if defined?(Datadog::Core::Configuration::Components)
  # monkey patch or extension
end
```

**Risk assessment:**
- Class is not public API
- Not returned by any public methods
- Mainly internal infrastructure
- Lower probability, but possible in test code

**Compatibility shim:**
```ruby
# lib/datadog/core/configuration/components.rb
module Datadog
  module Core
    module Configuration
      # Compatibility shim
      Components = ::Datadog::Components
    end
  end
end
```

---

## 2. Method Relocations - Public API Surface

### 🔴 **HIGH RISK: Public methods on `Datadog` module**

These are **definitely called** by customer code:

#### `Datadog.configuration` (moving from Core::Configuration to Datadog directly)
**Current:** Extended from Core::Configuration
**After:** Defined directly on Datadog
**Risk:** NONE - same location from caller perspective
✅ No breakage - public API unchanged

#### `Datadog.configure` (logic moving to ComponentLifecycle)
**Current:** Extended from Core::Configuration
**After:** Extended from ComponentLifecycle
**Risk:** NONE - same location from caller perspective
✅ No breakage - public API unchanged

#### `Datadog.shutdown!` (moving to ComponentLifecycle)
**Current:** Extended from Core::Configuration
**After:** Extended from ComponentLifecycle
**Risk:** NONE - same location from caller perspective
✅ No breakage - public API unchanged

#### `Datadog.logger` (moving to ComponentLifecycle)
**Current:** Extended from Core::Configuration
**After:** Extended from ComponentLifecycle
**Risk:** NONE - same location from caller perspective
✅ No breakage - heavily used, but public API unchanged

#### `Datadog.health_metrics` (moving to ComponentLifecycle)
**Current:** Extended from Core::Configuration (tagged @public_api)
**After:** Extended from ComponentLifecycle
**Risk:** NONE - same location from caller perspective
✅ No breakage - public API unchanged

---

## 3. Internal Method Relocations

### 🟡 **MEDIUM RISK: Protected methods**

#### `Datadog.components` (protected, moving to ComponentLifecycle)
**Current usage:**
```ruby
# Might be called by extension gems
Datadog.send(:components)
```

**Risk assessment:**
- Protected, so requires `send`
- Test helpers might use this
- Extension gems might reach in

**Compatibility:** Automatic via extend
✅ No shim needed - location unchanged from caller perspective

---

### 🟢 **LOW RISK: Private methods**

These are all private and moving to ComponentLifecycle:
- `reset!`
- `safely_synchronize`
- `build_components`
- `replace_components!`
- `logger_without_components`
- `handle_interrupt_shutdown!`
- `components?`
- `configuration?`
- `logger_without_configuration`

**Risk assessment:**
- Require `send` to call
- Internal implementation details
- Very low likelihood of external usage

**Compatibility:** Automatic via extend
✅ No shim needed - `Datadog.send(:reset!)` still works

---

## 4. Module References

### 🟡 **MEDIUM RISK: Core::Configuration module itself**

**Potential external usage:**
```ruby
# Extension library extending the module
module MyExtension
  include Datadog::Core::Configuration
end

# Checking if module is defined
if defined?(Datadog::Core::Configuration)
  # ...
end

# Accessing constants
Datadog::Core::Configuration::COMPONENTS_WRITE_LOCK
```

**Risk assessment:**
- Module remains, just loses methods
- Constants (COMPONENTS_WRITE_LOCK) moving to ComponentLifecycle
- Anyone using constants directly will break

**Compatibility for constants:**
```ruby
# lib/datadog/core/configuration.rb
module Datadog
  module Core
    module Configuration
      # Compatibility shims for moved constants
      COMPONENTS_WRITE_LOCK = ::Datadog::ComponentLifecycle::COMPONENTS_WRITE_LOCK
      COMPONENTS_READ_LOCK = ::Datadog::ComponentLifecycle::COMPONENTS_READ_LOCK
    end
  end
end
```

---

## 5. ComponentLifecycle - New Module

### 🟢 **LOW RISK: New module, no backward compat needed**

This is a new module, so no backward compatibility concerns.

---

## 6. Settings Extension Pattern

### 🔴 **MEDIUM-HIGH RISK: Product settings modules**

**Current:**
```ruby
# In customer code or extension gems
module MyCustomProduct
  module Configuration
    module Settings
      settings :my_custom do
        option :enabled
      end
    end
  end
end

# Then somewhere:
Datadog::Core::Configuration::Settings.extend(MyCustomProduct::Configuration::Settings)
```

**After refactor:**
```ruby
# Must now extend top-level Settings
Datadog::Settings.extend(MyCustomProduct::Configuration::Settings)
```

**Risk:** Extensions won't work without update

**Compatibility shim:**
```ruby
# lib/datadog/core/configuration/settings.rb
module Datadog
  module Core
    module Configuration
      Settings = ::Datadog::Settings

      # Could add deprecation warning when accessed:
      def self.const_missing(name)
        if name == :Settings
          warn "[DEPRECATION] Datadog::Core::Configuration::Settings is deprecated. Use Datadog::Settings instead."
          ::Datadog::Settings
        else
          super
        end
      end
    end
  end
end
```

---

## 7. RBS Type Signatures

### 🟡 **MEDIUM RISK: Type definitions**

**Current:**
```ruby
# In sig files
sig { returns(Datadog::Core::Configuration::Settings) }
def configuration; end
```

**After:**
```ruby
sig { returns(Datadog::Settings) }
def configuration; end
```

**Risk:** Type checkers in downstream libraries break

**Mitigation:**
- Update RBS files in dd-trace-rb
- Document type signature changes in CHANGELOG
- Downstream libraries need to update their types

---

## Summary Table: Breakage Likelihood & Mitigation

| Item | Old Location | New Location | Break Risk | Shim Needed? | Shim Complexity |
|------|--------------|--------------|------------|--------------|-----------------|
| Settings class | Core::Configuration::Settings | Datadog::Settings | 🔴 HIGH | ✅ YES | Simple const alias |
| Components class | Core::Configuration::Components | Datadog::Components | 🟡 MEDIUM | ✅ YES | Simple const alias |
| Public methods on Datadog | Various | Various | 🟢 NONE | ❌ NO | N/A - unchanged |
| Core::Configuration module | Same | Same (fewer methods) | 🟡 MEDIUM | ⚠️ PARTIAL | Const aliases for locks |
| ComponentLifecycle module | N/A (new) | Top-level | 🟢 NONE | ❌ NO | N/A - new |
| Settings.extend pattern | Core::Configuration::Settings | Datadog::Settings | 🔴 MEDIUM-HIGH | ✅ YES | Const alias handles it |
| RBS types | Various | Various | 🟡 MEDIUM | ❌ NO | Document in CHANGELOG |

---

## Recommended Compatibility Layer

### File: `lib/datadog/core/configuration/settings.rb`
```ruby
# After moving main content out, leave this:

require_relative '../../settings'  # Load new location

module Datadog
  module Core
    module Configuration
      # Backward compatibility: alias to new location
      Settings = ::Datadog::Settings

      # Optional: Add deprecation warning
      class << self
        def const_missing(name)
          if name == :Settings
            warn "[DEPRECATION] Datadog::Core::Configuration::Settings is deprecated. " \
                 "Use Datadog::Settings instead. " \
                 "This compatibility shim will be removed in version X.Y.Z"
            ::Datadog::Settings
          else
            super
          end
        end
      end
    end
  end
end
```

### File: `lib/datadog/core/configuration/components.rb`
```ruby
# After moving main content out, leave this:

require_relative '../../components'  # Load new location

module Datadog
  module Core
    module Configuration
      # Backward compatibility: alias to new location
      Components = ::Datadog::Components
    end
  end
end
```

### File: `lib/datadog/core/configuration.rb`
```ruby
# Add after ComponentLifecycle is loaded:

module Datadog
  module Core
    module Configuration
      # Backward compatibility: constants moved to ComponentLifecycle
      COMPONENTS_WRITE_LOCK = ::Datadog::ComponentLifecycle::COMPONENTS_WRITE_LOCK
      COMPONENTS_READ_LOCK = ::Datadog::ComponentLifecycle::COMPONENTS_READ_LOCK
    end
  end
end
```

---

## Testing Strategy for Compatibility

### 1. Test old constant access
```ruby
# spec/datadog/core/configuration/compatibility_spec.rb
RSpec.describe 'Backward compatibility' do
  it 'allows access to Settings via old constant' do
    expect(Datadog::Core::Configuration::Settings).to eq(Datadog::Settings)
  end

  it 'allows access to Components via old constant' do
    expect(Datadog::Core::Configuration::Components).to eq(Datadog::Components)
  end

  it 'allows access to COMPONENTS_WRITE_LOCK via old location' do
    expect(Datadog::Core::Configuration::COMPONENTS_WRITE_LOCK)
      .to eq(Datadog::ComponentLifecycle::COMPONENTS_WRITE_LOCK)
  end
end
```

### 2. Test external extension pattern
```ruby
# spec/datadog/core/configuration/extension_compatibility_spec.rb
RSpec.describe 'Extension compatibility' do
  it 'allows extending Settings via old constant name' do
    module TestExtension
      settings :test do
        option :enabled
      end
    end

    # Should work with old constant
    expect {
      Datadog::Core::Configuration::Settings.extend(TestExtension)
    }.not_to raise_error

    expect(Datadog.configuration.test.enabled).to eq(false)
  end
end
```

---

## CHANGELOG Entry Recommendations

```markdown
## [X.Y.Z] - YYYY-MM-DD

### Changed

**Internal refactoring: Configuration architecture**

We've refactored the internal configuration architecture to better separate concerns. While we've maintained backward compatibility, some internal constant locations have changed:

- `Datadog::Core::Configuration::Settings` → `Datadog::Settings` (compatibility alias provided)
- `Datadog::Core::Configuration::Components` → `Datadog::Components` (compatibility alias provided)

**Action required only if:**
- You're extending dd-trace-rb with custom configuration modules
- You're using internal constants directly (not recommended)
- You're using type signatures that reference the old class names

**Public API unchanged:** All public methods (`Datadog.configure`, `Datadog.configuration`, etc.) work exactly as before.

**Deprecation timeline:** Compatibility aliases will be removed in version X.Y.Z (estimated DATE).

### Deprecated

- `Datadog::Core::Configuration::Settings` constant (use `Datadog::Settings`)
- `Datadog::Core::Configuration::Components` constant (use `Datadog::Components`)
```

---

## Final Risk Assessment

**Overall Compatibility Risk: LOW-MEDIUM**

✅ **Public API completely unchanged** - Vast majority of users unaffected
✅ **Simple constant aliases handle most breakage** - Low implementation cost
⚠️ **Extension libraries may need updates** - Document clearly
⚠️ **Type signatures need updating** - Affects static analysis users

**Recommendation:**
1. Implement all suggested compatibility shims
2. Add comprehensive compatibility tests
3. Clearly document in CHANGELOG with migration guide
4. Consider 1-2 version grace period before removing shims
5. Add deprecation warnings to help users migrate

**The refactoring is safe to proceed with proper compatibility layer.**
