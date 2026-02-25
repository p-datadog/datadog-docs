# Tracer Repositories: Patch Release Documentation Analysis

**Analysis Date**: 2026-02-25
**Repositories Analyzed**: dd-trace-java, dd-trace-js, dd-trace-go, dd-trace-php, dd-trace-py, dtr (Ruby)

## Overview

This document summarizes the documentation about patch releases across all Datadog tracer repositories. It identifies what's well-documented, what's missing, and provides recommendations for improvement.

---

## dd-trace-java (Most Comprehensive)

### Documentation Location
- `docs/releases.md` - Release types and workflows
- `.github/workflows/README.md` - GitHub Actions documentation
- `.github/workflows/create-release-branch.yaml` - Automated branch creation

### Patch Release Definition
- **Trigger**: Done when needed only (not on a regular schedule)
- **Starting Point**: Minor release branch (`release/vM.N.x`)
- **Scope**: Important fixes only
- **Release Candidates**: Patch releases can be release candidates

### Release Types
1. **Minor release**: Default workflow, monthly cadence, ships latest from `master`
2. **Patch release**: On-demand, starts from `release/vM.N.x`, important fixes only
3. **Major release**: Exceptional, marks compatibility breaks
4. **Snapshot release**: Automatic for every PR

### Automated Workflow
When a minor release tag (e.g., `v1.54.0`) is pushed:
1. GitHub Actions automatically creates `release/vM.N.x` branch
2. System tests are pinned for that branch
3. PR is opened to merge the pinned tests into the release branch

### PR Management
- PRs can target `master` or patch release branches (`release/vM.N.x`)
- Milestones automatically attached to closed PRs
- Labels required: at least one `comp:` or `inst:` label + one `type:` label
- `tag: no release note` excludes PR from changelog

### Strengths
- Clear release type definitions
- Automated branch creation and setup
- Well-documented GitHub Actions workflows
- Clear milestone management

---

## dd-trace-js (Best Backporting Practices)

### Documentation Location
- `CONTRIBUTING.md` - Comprehensive contribution guidelines

### Core Philosophy: Backportability
> "We always backport changes from `master` to older versions to avoid release lines drifting apart and to prevent merge conflicts."

### Key Principles

#### Version Guards for Breaking Changes
Breaking changes must be guarded by version checks so they can land in `master` and be safely backported:

```javascript
const { DD_MAJOR } = require('./version')
if (DD_MAJOR >= 6) {
  // New behavior for v6+
} else {
  // Old behavior for v5 and earlier
}
```

#### Semantic Versioning Labels
- **semver-patch**: Bug or security fixes that don't alter existing behavior except to correct the specific issue
- **semver-minor**: New functionality that doesn't change existing behavior
- **semver-major**: Changes that break existing functionality

#### Release Target Labels
- `dont-land-on-vN.x`: Indicates which major release lines a PR should NOT land in
- Required for all `semver-major` PRs
- Allows branch-diff tooling to work smoothly

### Best Practices
1. **Keep changes small and incremental** for better backportability
2. **Test on previous branches**: Cherry-pick changes to previous vN.x branches locally to verify they merge cleanly
3. **Guard breaking changes** rather than creating separate implementations
4. **Avoid language/runtime features** that are too new for older supported versions

### Development Workflow
- Currently no CI to test PR mergeability to past release lines
- Manual testing recommended: cherry-pick to previous vN.x branches locally
- Node.js 18.0.0 is the baseline for ECMAScript features

### Strengths
- Excellent backporting philosophy and guidelines
- Clear semantic versioning practices
- Practical code examples for version guards
- Focus on keeping changes backport-friendly from the start

---

## dd-trace-go

### Documentation Location
- `CONTRIBUTING.md` - General contribution guidelines
- `scripts/autoreleasetagger/README.md` - Auto-tagging tool documentation

### Release Tooling

#### Auto Release Tagger
Purpose: Ensures all modules are tagged correctly for release, respecting dependency order.

Features:
- Automatic versioning based on existing tags
- Nested module support
- Dependency-aware tagging (dependencies tagged before dependents)

Usage:
```sh
# Dry run to check for issues
go run ./scripts/autoreleasetagger -dry-run -root ../..

# Tag without pushing
go run ./scripts/autoreleasetagger -disable-push -root ../..

# Full release
go run ./scripts/autoreleasetagger -root ../..
```

### Module Management
- Repository has nested modules for contrib packages
- Reduces dependency surface to strictly required dependencies
- Complex multi-module versioning

### PR Conventions
- Follow conventional commits format: `<type>(scope): <description>`
- Examples:
  - `feat(contrib/http): add support for custom headers`
  - `fix(ddtrace/tracer): resolve memory leak in span processor`

### Release Process
- References internal release checklist (Confluence)
- Version must be updated before tagging
- Tool handles dependency ordering automatically

### Strengths
- Sophisticated multi-module release tooling
- Automated dependency-aware tagging
- Clean versioning strategy

### Gaps
- No public documentation on patch release workflow
- Release checklist is internal only
- Limited guidance on when to create patch vs. minor release

---

## dd-trace-php

### Documentation Location
- `CONTRIBUTING.md` - Basic contribution guidelines
- `dockerfiles/release-candidates/README.md` - RC Docker images

### Development Setup
- Docker-based development with PHP version-specific containers
- Uses Composer for dependency management
- PSR-2 coding style enforced with PHP_CodeSniffer

### Testing
- Comprehensive test suites (unit, integration, auto-instrumentation, web frameworks)
- Snapshot testing for tracer output validation
- C extension tests separate from PHP tests

### CI/CD
- CircleCI for automated checks
- All checks must pass before PR merge

### Strengths
- Good development environment documentation
- Comprehensive testing infrastructure

### Gaps
- **No patch release documentation**
- No versioning or release workflow information
- No backporting guidelines
- Release process appears to be undocumented or internal only

---

## dd-trace-py (Python)

### Documentation Location
- `.claude/skills/releasenote/SKILL.md` - Release notes skill documentation
- `AGENTS.md` - Project structure and guidelines
- `docs/releasenotes.rst` - Release note conventions (referenced)

### Release Notes Infrastructure: Reno

#### Process
1. Create release note fragment for each change/PR:
   ```bash
   riot run reno new <title-slug>
   ```

2. Edit generated YAML fragment:
   ```yaml
   features:
     - |
       Description of the new feature

   fixes:
     - |
       Description of the bug fix
   ```

3. Fragments stored in `releasenotes/notes/`

#### Categories
- `features` - New functionality
- `fixes` - Bug fixes
- `deprecation` - Deprecated features
- `breaking-change` - Breaking changes
- `misc` - Miscellaneous changes

#### Best Practices
- Start with action verb: *Add…*, *Fix…*, *Improve…*, *Deprecate…*
- Reference PR/issue numbers when relevant
- User-facing language (avoid internal terminology)
- One fragment per change/PR unless explicitly belongs in existing note

### Versioning Policy
- References versioning policy with maintenance mode definitions
- 2.x line entered maintenance mode with 3.0.0 release
- Maintenance mode = bugfix changes on last few minor releases only

### Strengths
- Well-structured release notes process
- Clear tooling (Reno/riot)
- Good documentation for release note creation

### Gaps
- No patch release workflow documentation
- No guidance on backporting or cherry-picking
- No criteria for when to cut a patch vs. minor release

---

## dtr (Ruby)

### Documentation Location
- `CONTRIBUTING.md` - General contribution guidelines
- `CLAUDE.md` - AI assistant rules
- `.github/PULL_REQUEST_TEMPLATE.md` - PR template

### PR Requirements
- **ALWAYS** push branches to `DataDog/dd-trace-rb` (not forks)
- **ALWAYS** use `--repo DataDog/dd-trace-rb` with gh commands
- Use PR template as starting point
- Write for code reviewers (concise)
- Add `--label "AI Generated"` when creating AI-generated PRs

### Changelog Conventions
- Changelog entries are for **customers only**
- Consider changes from user/customer perspective
- Internal changes (telemetry, CI, tooling) = "None" for changelog

### Development Workflow
- Docker-based development: `docker compose run --rm tracer-3.4 /bin/bash`
- Bundle, rake, and rspec for testing
- StandardRB for code formatting
- Steep for type checking (RBS)

### Code Change Rules
- Read files before editing
- "Suggest" = analyze only, don't modify
- "Fix/change/update" = make the changes
- Alert user if request contradicts code evidence

### Strengths
- Clear PR workflow and conventions
- Good customer-focused changelog guidance
- Well-documented development environment

### Gaps
- **No patch release documentation**
- No versioning or release workflow information
- No backporting guidelines
- Release process appears to be internal only

---

## dd-trace-dotnet (.NET)

### Notes from Analysis
While not in the primary analysis, CHANGELOG.md revealed:

### Major Release Cadence
- Yearly major releases starting from v3.0.0 (2024), v4.0.0 (2025), etc.
- Follows .NET's yearly major release cadence
- Focus: Support new .NET versions, remove EOL frameworks and operating systems
- Minimal breaking changes expected

### Philosophy
> "By adopting a yearly cycle for the .NET tracer, we are able to quickly react to any new requirements. It also ensures customers know when to expect major versions of the tracer."

---

## Cross-Repository Findings

### What's Well Documented

1. **Release Type Definitions** (Java)
   - Clear distinction between minor, patch, major, and snapshot releases
   - When each type is used

2. **Backporting Practices** (JavaScript)
   - Philosophy and rationale
   - Technical implementation patterns (version guards)
   - Semantic versioning labels

3. **Release Notes Infrastructure** (Python)
   - Tooling (Reno)
   - Process and conventions
   - Categories and best practices

4. **Automated Workflows** (Java)
   - Branch creation
   - System test pinning
   - Milestone management

5. **Multi-Module Releases** (Go)
   - Dependency-aware tagging
   - Automated tooling

### What's Missing Across All Repos

1. **Step-by-Step Patch Release Guides**
   - No repo has a complete "how to cut a patch release" document
   - Process appears to be tribal knowledge or internal documentation

2. **Decision Criteria**
   - When to create a patch release vs. wait for next minor
   - Severity levels for patch-worthy bugs
   - Customer impact assessment guidelines

3. **Backporting Procedures**
   - Cherry-picking workflows
   - Conflict resolution strategies
   - Testing requirements for backported changes

4. **Emergency Hotfix Procedures**
   - Fast-track release process
   - Approval requirements
   - Communication protocols

5. **Version Branch Maintenance**
   - How long to maintain release branches
   - When to deprecate/EOL a release line
   - Support commitments per version

6. **Testing Requirements**
   - What tests are required for patch releases
   - Regression testing scope
   - Performance validation

7. **Communication and Documentation**
   - Release note requirements for patches
   - Customer communication for critical fixes
   - Internal stakeholder notifications

---

## Recommendations

### For All Repositories

Create a dedicated **`RELEASING.md`** or expand **`docs/releases.md`** to include:

#### 1. Release Types and Criteria
```markdown
## Patch Releases

### When to Create a Patch Release
- Critical security vulnerabilities (CVSS > 7.0)
- Production-breaking bugs affecting multiple customers
- Data corruption or data loss issues
- Performance regressions > 20%
- Compliance or regulatory issues

### When to Wait for Minor Release
- Non-critical bugs with workarounds
- Feature enhancements
- Performance improvements < 10%
- Dependency updates without security implications
```

#### 2. Step-by-Step Process
```markdown
## Cutting a Patch Release

### Prerequisites
- [ ] Bug fix merged to master
- [ ] Fix approved for backport
- [ ] Release branch exists (release/vM.N.x)

### Steps
1. Cherry-pick fix to release branch:
   ```bash
   git checkout release/v1.54.x
   git cherry-pick <commit-sha>
   ```

2. Run regression test suite:
   ```bash
   <repo-specific test command>
   ```

3. Update version and changelog:
   - Increment patch version
   - Add release notes

4. Create and push tag:
   ```bash
   git tag v1.54.1
   git push origin v1.54.1
   ```

5. Monitor automated release process
6. Verify artifacts published
7. Update documentation and announce
```

#### 3. Backporting Guidelines
```markdown
## Backporting Changes

### Backport Checklist
- [ ] Change is backwards compatible
- [ ] No new dependencies introduced
- [ ] Tests pass on target version
- [ ] No breaking API changes
- [ ] Documentation updated if needed

### Version Guards (for breaking changes)
<Language-specific examples>

### Conflict Resolution
<Guidance on handling merge conflicts>
```

#### 4. Emergency Hotfix Procedures
```markdown
## Emergency Hotfix Process

### Severity Levels
- **Critical (P0)**: Production down, data loss, security breach
- **High (P1)**: Major functionality broken, significant customer impact
- **Medium (P2)**: Important but not blocking, workaround exists

### Fast-Track Approval
- P0: On-call engineer approval + notify team
- P1: Team lead approval
- P2: Follow normal patch release process

### Communication
- Create incident channel
- Notify customer support
- Post status updates every 30 minutes
- Post-mortem within 48 hours
```

### Repository-Specific Recommendations

#### dd-trace-java
- ✅ Already has good foundation
- Add: Backporting guidelines (borrow from JavaScript)
- Add: Severity criteria for patches
- Add: Emergency hotfix procedures

#### dd-trace-js
- ✅ Excellent backporting practices
- Add: Step-by-step patch release process
- Add: When to create patch vs. minor
- Consider: CI to test backport mergeability (mentioned as future work)

#### dd-trace-go
- Add: Public patch release documentation
- Add: When to use patch vs. minor releases
- Add: Backporting procedures for multi-module repo
- Document: Internal release checklist publicly (or create public version)

#### dd-trace-php
- **Critical**: Need full release documentation
- Add: Versioning strategy
- Add: Patch release process
- Add: Backporting guidelines

#### dd-trace-py
- ✅ Good release notes infrastructure
- Add: Patch release process
- Add: Backporting guidelines
- Add: Cherry-picking workflow

#### dtr (Ruby)
- **Critical**: Need full release documentation
- Add: Versioning strategy
- Add: Patch release process
- Add: Backporting guidelines

---

## Example: Comprehensive Release Documentation Template

```markdown
# Release Process

## Release Schedule
- **Minor Releases**: First week of every month
- **Patch Releases**: As needed for critical issues
- **Major Releases**: Annual or when breaking changes required

## Release Types

### Minor Release (vM.N.0)
- **Cadence**: Monthly
- **Content**: New features, enhancements, non-critical bug fixes
- **Branch**: Develop from `master`
- **Testing**: Full test suite + integration tests
- **Approval**: Team lead

### Patch Release (vM.N.P where P > 0)
- **Cadence**: On-demand
- **Content**: Critical bug fixes, security patches
- **Branch**: Cherry-pick to `release/vM.N.x`
- **Testing**: Regression suite + affected feature tests
- **Approval**: Varies by severity (see Emergency Procedures)

### Major Release (vM.0.0 where M > current)
- **Cadence**: Yearly or as needed
- **Content**: Breaking changes, major features
- **Branch**: Special release branch from `master`
- **Testing**: Full test suite + manual QA
- **Approval**: Engineering leadership + product

## Patch Release Criteria

### Create Patch Release For:
- [ ] Security vulnerabilities (any CVSS score)
- [ ] Production-breaking bugs
- [ ] Data corruption/loss risks
- [ ] Memory leaks or resource exhaustion
- [ ] Critical performance regressions (>20%)
- [ ] Compliance violations

### Wait for Next Minor For:
- [ ] Feature requests
- [ ] Enhancements to existing features
- [ ] Non-critical bugs with workarounds
- [ ] Documentation-only changes
- [ ] Internal refactoring
- [ ] Dependency updates (non-security)

## Step-by-Step: Patch Release

### 1. Preparation
```bash
# Ensure fix is in master first
git checkout master
git pull origin master

# Verify the fix commit
git log --oneline -10

# Note the commit SHA
COMMIT_SHA=<sha-of-fix>
```

### 2. Cherry-Pick to Release Branch
```bash
# Checkout release branch
git checkout release/v1.54.x
git pull origin release/v1.54.x

# Cherry-pick the fix
git cherry-pick $COMMIT_SHA

# Resolve conflicts if any
# <resolve conflicts>
# git add <resolved-files>
# git cherry-pick --continue
```

### 3. Test the Fix
```bash
# Run regression tests
<repo-specific test command>

# Run integration tests
<repo-specific integration tests>

# Manual verification if needed
```

### 4. Update Version and Changelog
```bash
# Update version file
<edit version file>

# Update CHANGELOG.md
<add release notes>

# Commit changes
git add <version-file> CHANGELOG.md
git commit -m "chore: Bump version to v1.54.1"
```

### 5. Tag and Push
```bash
# Create tag
git tag -a v1.54.1 -m "Release v1.54.1"

# Push branch and tag
git push origin release/v1.54.x
git push origin v1.54.1
```

### 6. Monitor Release Process
- [ ] Automated build completes successfully
- [ ] Artifacts published to package registry
- [ ] Docker images pushed
- [ ] Documentation updated
- [ ] GitHub release created

### 7. Communicate
- [ ] Notify customer support
- [ ] Update status page if applicable
- [ ] Send release announcement
- [ ] Close related issues

## Backporting Guidelines

### Compatible Changes (Safe to Backport)
- Bug fixes that don't change behavior
- Security patches
- Performance improvements
- Dependency updates (security)
- Documentation fixes

### Incompatible Changes (Requires Version Guard)
- API changes
- Behavior changes
- New features that affect existing code
- Breaking dependency updates

### Version Guard Pattern
<Language-specific example>

### Testing Backports
```bash
# Test on target version
<commands>

# Verify no regressions
<commands>
```

## Emergency Hotfix Procedures

### Severity Assessment

#### P0 - Critical (< 2 hour SLA)
- Production completely down
- Data loss or corruption
- Security breach

**Process**:
1. On-call engineer creates hotfix branch
2. Implements fix with another engineer
3. Minimal testing (automated + smoke test)
4. Deploy to staging
5. Deploy to production with monitoring
6. Tag and release
7. Post-mortem within 24 hours

#### P1 - High (< 24 hour SLA)
- Major functionality broken
- Significant customer impact
- High-severity security issue

**Process**:
1. Team lead approval required
2. Follow standard patch process with expedited review
3. Full testing required
4. Deploy during business hours if possible

#### P2 - Medium (< 1 week SLA)
- Important but not blocking
- Workaround exists
- Medium-severity security issue

**Process**:
1. Follow standard patch release process
2. Schedule for next patch window

## Version Branch Maintenance

### Support Policy
- **Current major version**: Full support
- **Previous major version**: Security and critical bugs only (12 months)
- **Older versions**: No support (EOL)

### Branch Lifecycle
1. **Active**: Receives all patches
2. **Maintenance**: Security and critical bugs only
3. **EOL**: No updates

### EOL Process
1. Announce EOL 6 months in advance
2. Send reminders at 3 months and 1 month
3. Archive branch after EOL date
4. Update documentation

## FAQ

### Q: Should I backport a performance improvement?
A: Only if it's a significant regression (>20%) introduced in recent versions.

### Q: How many release branches should we maintain?
A: Current major version + previous major version (12 months).

### Q: Can I backport a new feature?
A: No, new features go in minor/major releases only.

### Q: What if cherry-pick has conflicts?
A: Try to resolve. If extensive changes needed, re-implement the fix for that version.

## Tools and Scripts

### Release Script
```bash
./scripts/release.sh --type patch --version v1.54.1
```

### Backport Helper
```bash
./scripts/backport.sh <commit-sha> release/v1.54.x
```

### Version Checker
```bash
./scripts/check-version.sh
```
```

---

## Conclusion

While the tracer repositories have varying levels of release documentation, there's a clear opportunity to improve patch release processes across all repos. The best practices exist but are scattered across different repos:

- **Java** leads in automation and structure
- **JavaScript** leads in backporting philosophy
- **Python** leads in release notes infrastructure
- **Go** leads in multi-module versioning

By combining these strengths and filling the gaps identified, each repository can provide clear, comprehensive guidance for maintainers and contributors on how to handle patch releases effectively.

### Immediate Action Items

1. **High Priority** (PHP, Ruby): Create basic release documentation
2. **Medium Priority** (All): Add patch release decision criteria
3. **Medium Priority** (All): Document step-by-step patch release process
4. **Low Priority** (All): Add emergency hotfix procedures

### Long-Term Improvements

1. Cross-repo release playbook with language-specific variations
2. Automated backport tooling (learning from JavaScript and Go)
3. CI validation for backport compatibility
4. Shared release engineering knowledge base
