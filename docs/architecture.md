# GlobalFin Services — CI/CD Pipeline Architecture

This document describes the high-level architecture of our CI/CD pipelines using GitHub Actions.

## Key Features
- **Monorepo Strategy**: Single repository containing `/frontend`, `/backend`, `/infrastructure`, and `/docs`.
- **Selective Execution**: Path filtering to avoid triggering unnecessary jobs for documentation-only changes.
- **Reusable Workflows**: Standardized templates for testing, building, and validation.
- **Enterprise Hardening**: Strictly defined permissions, pinned action versions (SHA hashes), environment limits, and OIDC authentication.
