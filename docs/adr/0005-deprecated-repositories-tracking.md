# Deprecated Repositories Tracking Strategy

## Status

Proposed

### Date

2025-01-16

### Owner

[Denis Filatov](https://github.com/denifilatoff)

### Participants and approvers

- [pankratovsa](https://github.com/pankratovsa)
- [shumnic](https://github.com/shumnic)
- [Alexey Karasev](https://github.com/asatt)
- [IldarMinaev](https://github.com/IldarMinaev)
- [Vladimir Sitnikov](https://github.com/vlsi)
- [egbu](https://github.com/egbu)

### Related ADRs

None

## Context

Qubership platform consists of multiple repositories across different services and components. Over time, some repositories become deprecated or replaced by newer implementations. Without a clear strategy for tracking and managing these deprecated repositories, it becomes difficult for developers and users to understand the current state of the codebase and identify actively maintained components.

There is a need for a standardized approach to mark repositories as deprecated, provide clear migration paths, and maintain visibility of archived projects while preventing confusion about their maintenance status.

## Decision

We will implement a standardized approach for tracking and managing deprecated repositories in Qubership, following the Google Archive pattern: [googlearchive](https://github.com/googlearchive).

### Implementation Strategy

1. **Archive Organization**: Create a separate `netcracker-archive` organization to house deprecated repositories
2. **Repository Status**: Set deprecated repositories to "Public archive" status in GitHub
3. **Standardized Badge**: Add a status badge `![Project Status](https://img.shields.io/badge/status-archived-red)` at the top of `README.md` file
4. **Project Status Section**: Add a mandatory "Project Status" H1 section in `README.md` with:
   - Clear statement about maintenance status
   - Replacement or migration information
   - Link to actively maintained alternatives

### Standard `README.md` Template for Archived Projects

```markdown
![Project Status](https://img.shields.io/badge/status-archived-red)

# Project Status

This project is no longer actively maintained, and remains here as an archive of this work.

For a replacement, check out [actively maintained alternative](link-to-replacement).

## [Original Project Title]

[Rest of original `README.md` content]
```

### Best Practices & Statements

- Follow GitHub's repository archiving guidelines
- Maintain clear documentation trails for migration paths
- Preserve historical context for future reference

### Justification

- **Clarity**: Developers can immediately identify repository status
- **Consistency**: Standardized approach across all Qubership repositories
- **Migration Support**: Clear paths for users to find replacements
- **Reduced Confusion**: Prevents accidental usage of deprecated components
- **Industry Standard**: Following proven pattern (example: [googlearchive](https://github.com/googlearchive/), [MicrosoftArchive](https://github.com/MicrosoftArchive), [amazon-archives](https://github.com/amazon-archives)

Alternatives considered:

- Simply marking repositories as private (rejected due to loss of historical visibility)
- Adding only notices in `README.md` without organizational separation (rejected due to lack of clear separation)
- Making public archive within main [Netcracker organization](https://github.com/Netcracker) (rejected due to further growth of repository count in main organization)
- Deleting outdated repositories entirely (rejected due to loss of historical context)

## Consequences

**Positive:**

- Clear visibility of repository maintenance status
- Standardized user experience across Qubership ecosystem
- Preserved historical context and migration paths
- Reduced support burden for deprecated components

**Negative:**

- Additional overhead for repository maintenance and migration
- Need to maintain archive organization and processes
- Potential confusion during transition period

**Neutral:**

- Requires team training on new archiving procedures
- Need to establish governance for archive decisions
