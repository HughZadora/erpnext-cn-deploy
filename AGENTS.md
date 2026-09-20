# Agent Instructions

## Scope

This file applies to the entire `erpnext-cn-deploy` repository.

## Product boundary

This repository owns deployment and operating procedures for an ERPNext installation based on
`frappe_docker`. Repository edits are not permission to operate a live ERPNext server.

## Working rules

- Treat the README, Makefile, `config/`, `scripts/`, and deployment/operations documentation as the
  repository's source of truth.
- Prefer upstream Frappe/ERPNext and Docker mechanisms before adding custom deployment machinery.
- Verify reported problems before changing deployment instructions or configuration.
- Keep repository changes separate from production actions. Starting, stopping, upgrading,
  migrating, restoring, pruning, or otherwise mutating a live installation requires the applicable
  operational authorization and backup/recovery readiness.
- Preserve existing business-operation documentation unless the task specifically changes it.
- Do not introduce a second orchestration framework, state database, or generic repository standard.
- Never commit real passwords, tokens, production `.env` values, backups, customer data, or other
  sensitive deployment material.

## Validation

Run the project-local static verifier before claiming a repository change is complete:

```sh
./scripts/repository-check
```

This check validates repository-owned files and syntax only. It deliberately does not connect to or
modify a live ERPNext installation; production verification is a separate operational gate.
