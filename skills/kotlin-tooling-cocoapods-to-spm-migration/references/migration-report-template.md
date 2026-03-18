# Migration Report Template

After migration, write `MIGRATION_REPORT.md` in the project root.

Use this structure:

```markdown
# Migration Report: CocoaPods to SwiftPM Import

**Project:** <project name>
**Module migrated:** <module name>
**Date:** <YYYY-MM-DD>
**Kotlin version:** <old version> -> <new version>
**Status:** <Completed successfully | Completed with workarounds | Failed>

## Pre-Migration State

### CocoaPods Dependencies

| Pod | Version | Mode | Notes |
| --- | --- | --- | --- |
| <PodName> | <version> | Regular / linkOnly | <notes> |

### Framework Configuration

- `baseName`: <value>
- `isStatic`: <before> -> <after>
- Deployment target: <value>

### Kotlin Files Using `cocoapods.*`

| File | Imports |
| --- | --- |
| <path> | `cocoapods.<Module>.<Class>` |

### Non-KMP CocoaPods

<List them or state "None">

### Atypical Project Configuration

<Describe unusual build logic, disabled EmbedAndSign tasks, custom podspec or
linker configuration, or other migration-relevant details.>

## Migration Steps

### Phase 2: Gradle Configuration

<Describe Kotlin version or repository changes. Include snippets for
non-trivial edits.>

### Phase 3: `swiftPMDependencies`

<Include the final block and explain notable choices such as
`discoverModulesImplicitly`, `importedModules`, and `isStatic`.>

### Phase 4: Import Transformations

| File | Before | After | Source |
| --- | --- | --- | --- |
| <path> | `cocoapods.X.Y` | `swiftPMImport...Y` | swiftPMImport |
| <path> | `cocoapods.X.Y` | `cocoapods.X.Y` | bundled klib |

### Phase 5: iOS Project Reconfiguration

<Document integration tasks, sandboxing updates, dSYM script changes, and any
manual Xcode edits.>

### Phase 6: CocoaPods Removal

<List the plugin, block, files, and build logic removed.>

### Phase 7: Verification

<Record the commands run and their results.>

## Errors Encountered

### Error #1: <Short title>

**Phase:** <phase>
**Symptom:** <exact error or behavior>
**Root cause:** <analysis>
**Fix:** <resolution>
**Generalizable:** <Yes/No>

## Non-Trivial Decisions

<Explain the decisions that required judgment, especially preserved imports,
framework linkage, or Xcode integration trade-offs.>

## Files Changed

### Gradle Files
- <path> — <what changed>

### Kotlin Sources
- <path> — <what changed>

### Xcode Project Files
- <path> — <what changed>

### Created
- <path> — <what it is>

### Deleted
- <path> — <what it was>
```

## Writing Guidelines

- Be specific about files, modules, commands, and errors.
- Include before/after snippets when the change is not obvious.
- Clearly mark `cocoapods.*` imports that were intentionally preserved.
- Separate migration errors from pre-existing build issues when possible.
- Keep the structure stable so the report is readable by both humans and tools.
