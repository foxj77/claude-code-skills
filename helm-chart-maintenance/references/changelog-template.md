# Changelog Template

## Format

Based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) and [Semantic Versioning](https://semver.org/).

## CHANGELOG.md Template

```markdown
# Changelog

All notable changes to this chart will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- New features that have been added

### Changed
- Changes in existing functionality

### Deprecated
- Features that will be removed in future versions

### Removed
- Features that have been removed

### Fixed
- Bug fixes

### Security
- Security fixes and improvements

## [2.0.0] - 2024-01-15

### Added
- Added support for HorizontalPodAutoscaler
- Added values.schema.json for input validation
- Added ServiceMonitor for Prometheus integration

### Changed
- **BREAKING**: Renamed `image.name` to `image.repository`
- **BREAKING**: Minimum Kubernetes version is now 1.23
- Updated default resource limits
- Improved security context defaults

### Removed
- **BREAKING**: Removed deprecated `legacyMode` option

### Security
- Container now runs as non-root by default
- Added Pod Security Standards support

## [1.5.0] - 2023-12-01

### Added
- Added `extraEnv` and `extraVolumes` options
- Added topology spread constraints support

### Changed
- Updated PostgreSQL dependency to 12.x

### Deprecated
- `legacyMode` is deprecated and will be removed in v2.0.0

### Fixed
- Fixed ingress path configuration for Kubernetes 1.24+

## [1.4.2] - 2023-11-15

### Fixed
- Fixed service port configuration when using NodePort
- Fixed incorrect label selector in deployment

## [1.4.1] - 2023-11-01

### Security
- Updated base image to patch CVE-2023-XXXXX

## [1.4.0] - 2023-10-15

### Added
- Added configurable liveness and readiness probes
- Added support for custom annotations

### Changed
- Improved NOTES.txt output

## [1.3.0] - 2023-09-01

### Added
- Initial public release

[Unreleased]: https://github.com/org/chart/compare/v2.0.0...HEAD
[2.0.0]: https://github.com/org/chart/compare/v1.5.0...v2.0.0
[1.5.0]: https://github.com/org/chart/compare/v1.4.2...v1.5.0
[1.4.2]: https://github.com/org/chart/compare/v1.4.1...v1.4.2
[1.4.1]: https://github.com/org/chart/compare/v1.4.0...v1.4.1
[1.4.0]: https://github.com/org/chart/compare/v1.3.0...v1.4.0
[1.3.0]: https://github.com/org/chart/releases/tag/v1.3.0
```

## Change Types

| Type | Description | Version Impact |
|------|-------------|----------------|
| Added | New features | MINOR |
| Changed | Changes in existing functionality | MINOR or MAJOR |
| Deprecated | Soon-to-be removed features | MINOR |
| Removed | Removed features | MAJOR |
| Fixed | Bug fixes | PATCH |
| Security | Security fixes | PATCH or MINOR |

## Breaking Change Indicators

Always prefix breaking changes:
- `**BREAKING**:` in the changelog
- Document migration path
- Consider major version bump

```markdown
### Changed
- **BREAKING**: Renamed `image.name` to `image.repository`

  Migration:
  ```yaml
  # Before
  image:
    name: myapp

  # After
  image:
    repository: myapp
  ```
```

## ArtifactHub Annotations

```yaml
# Chart.yaml
annotations:
  artifacthub.io/changes: |
    - kind: added
      description: Support for HorizontalPodAutoscaler
    - kind: changed
      description: Renamed image.name to image.repository
      links:
        - name: Migration guide
          url: https://github.com/org/chart/blob/main/MIGRATION.md
    - kind: fixed
      description: Fixed service port configuration
    - kind: security
      description: Updated base image for CVE-2023-XXXXX
```
