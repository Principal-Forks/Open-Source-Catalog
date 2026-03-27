# NASA Open Source Catalog Architecture

## Overview

The NASA Open Source Catalog serves as the authoritative registry for NASA's open source software portfolio, managing metadata, ensuring federal compliance, and enabling public discovery of NASA's contributions to open source.

## Problem Statement

NASA needs a centralized system to:
- Track and catalog all open source projects across NASA centers
- Maintain compliance with federal code sharing mandates (M-16-21 policy)
- Provide a public-facing portal for developers to discover NASA projects
- Enable community contributions while maintaining data quality
- Preserve version history for audit and compliance purposes

## Architecture Components

### Data Submission Layer

The catalog accepts project submissions through three channels:

1. **GitHub Pull Requests**: Community members and NASA developers can submit projects directly via PRs, enabling transparent, version-controlled contributions.

2. **Web Submission Form**: An internal web application (code-submission-app.code.nasa.gov) provides a user-friendly interface for NASA staff to submit projects without requiring Git knowledge.

3. **Robot Branch**: Automated submissions from internal NASA systems are merged via a dedicated robot branch, enabling programmatic updates.

### Data Storage Layer

The system maintains three JSON-based data stores in a Git repository:

1. **catalog.json**: A community-friendly format containing simplified project metadata. This format prioritizes readability and ease of contribution, with fields like NASA Center, Software name, Public Code Repo, Description, License, Categories, and Contributors.

2. **code.json**: A federal compliance format adhering to the code.gov schema specification. This comprehensive format includes additional fields required by federal policy, such as structured release metadata, permissions, usage types, labor hours, development status, and AI-generated keywords.

3. **code-history.json**: A version history archive that preserves UUID-keyed snapshots of projects over time, enabling audit trails, rollback capabilities, and compliance verification.

### Validation and Quality Assurance

**Schema Validation**: All submissions are validated against code-nasa-schema.json, a JSON Schema (draft-04) specification that ensures data quality and federal compliance. The validator checks:
- Required fields (name, repositoryURL, description, contact, laborHours, tags, permissions)
- Data type correctness
- URL format validity
- License information completeness

**Travis CI Pipeline**: Continuous integration automatically runs on every commit to:
- Execute schema validation
- Verify data integrity
- Run quality checks
- Gate publication to external systems

### Publication Layer

Once validated, project data is published to two external platforms:

1. **code.nasa.gov**: The public-facing NASA open source portal where developers can browse, search, and discover NASA projects. This portal provides a user-friendly interface built on WordPress.

2. **code.gov**: The federal government's code sharing platform, which aggregates open source projects from all federal agencies. NASA's catalog feeds into this central registry to fulfill federal transparency requirements.

## Design Decisions

### Why Flat File JSON Instead of a Database?

The system uses JSON files in a Git repository rather than a traditional database for several reasons:
- **Auditability**: Git provides built-in version control and audit trails
- **Transparency**: Anyone can inspect the catalog's history and evolution
- **Simplicity**: No database infrastructure to maintain or scale
- **Portability**: Data can be easily replicated, forked, and backed up
- **Community**: GitHub's collaboration features enable community contributions

### Dual-Format Data Model

The system maintains two separate JSON formats (catalog.json and code.json) because:
- **catalog.json** optimizes for community contribution with simple, intuitive fields
- **code.json** satisfies complex federal compliance requirements
- This separation prevents overwhelming contributors with bureaucratic complexity while still meeting regulatory obligations

### Schema-Driven Validation

Using JSON Schema provides:
- **Clear contracts**: Contributors know exactly what fields are required
- **Automated enforcement**: Validation runs automatically in CI/CD
- **Self-documentation**: The schema serves as living documentation
- **Tooling integration**: Many tools can generate forms and validation from JSON Schema

## Workflow Patterns

### Submitting a New Project

1. Contributor submits project via GitHub PR, web form, or automated system
2. Submission enters the master branch as a Git commit
3. Data is written to both catalog.json and code.json formats
4. A historical snapshot is stored in code-history.json
5. Travis CI runs schema validation
6. On successful validation, the project is published to code.nasa.gov and code.gov
7. The project becomes publicly discoverable

### Updating an Existing Project

1. Contributor updates project information via PR or web form
2. Changes are validated against the schema
3. code-history.json preserves the previous version with UUID key
4. Updated data propagates to public portals
5. Version history enables rollback if needed

## Error Scenarios and Recovery

### Schema Validation Failure

**Scenario**: A submission violates the JSON Schema (e.g., missing required field, invalid URL format)

**Recovery**:
- Travis CI fails the build and blocks publication
- Validation error message identifies the specific issue
- Contributor fixes the issue and resubmits
- No invalid data reaches public portals

### Merge Conflict

**Scenario**: Multiple submissions modify the same project simultaneously

**Recovery**:
- Git merge conflict prevents automatic merge
- Repository maintainers manually resolve the conflict
- Git history preserves all attempted changes
- Final resolution is explicitly committed

### Data Corruption

**Scenario**: A bad commit introduces malformed JSON or breaks the schema

**Recovery**:
- Git history enables instant rollback to previous working state
- code-history.json provides backup snapshots
- Schema validation catches corruption before publication
- Affected data can be surgically repaired or restored

## Compliance and Governance

The system ensures compliance with:
- **M-16-21 Federal Source Code Policy**: Requires agencies to release at least 20% of new custom code as open source
- **code.gov Schema Specification**: Mandates specific metadata fields for federal agency code inventories
- **NASA Open Source Agreement (NOSA) 1.3**: The default license for NASA open source projects
- **OCIO and HQ Mission Requirements**: Tracks projects by NASA center and mission directorate

## Future Considerations

Potential enhancements to the system could include:
- Automated GitHub repository metrics collection (stars, forks, issues)
- Integration with NASA's Software Engineering Practices database
- Real-time synchronization with code.nasa.gov instead of batch publication
- Enhanced search and filtering capabilities using Elasticsearch
- GraphQL API for programmatic access to catalog data
- Automated license compatibility checking
- Dependency vulnerability scanning integration
