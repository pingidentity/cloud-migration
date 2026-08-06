# cloud-migration

Public assets for the [cloud.pingmigrate.com](https://cloud.pingmigrate.com) tool — Ping Identity's
migration path for moving PingFederate configuration into PingOne. This repo is served as a static
site (via GitHub Pages) and hosts the downloadable scripts, install packages, and images referenced
by the migration tool and its documentation.

## What's here

| Path | Purpose |
| - | - |
| [scripts/](scripts/) | Terraform-driven migration tooling and PingOne client/secret helpers |
| [scripts/prod/](scripts/prod/) | PingFederate export scripts run on the source PF host |
| [downloads/](downloads/) | Distributable Client Migration Tool package and install docs |
| [images/](images/) | Screenshots/diagrams used in migration documentation |

## Migration workflow

1. **Export** — run [scripts/prod/cloud-migration-pingfederate-export.sh](scripts/prod/cloud-migration-pingfederate-export.sh)
   (or the older [cloud-migration.sh](scripts/prod/cloud-migration.sh)) on the PingFederate admin
   host. It pulls OAuth clients, IdP/SP connections, adapters, auth policies, and signing keys via
   the PF Admin API, encrypts the exported signing certs with a migration key you provide, and
   packages everything into a dated `pingfederate-<timestamp>.zip`.
2. **Apply** — feed that export into either:
   - [scripts/cloud-migration-pingcli.sh](scripts/cloud-migration-pingcli.sh) — applies a pingcli-based
     JSON export (produced by pingmigrate's "Download pingcli Zip" button) directly to a PingOne
     environment via [pingcli](https://github.com/pingidentity/pingcli). Supports `--delete-all` to
     roll back everything it created, tracked in a per-environment manifest file.
   - [scripts/cloud-migration-pingone.sh](scripts/cloud-migration-pingone.sh) — the Terraform-based
     path; wraps `terraform init/plan/apply/destroy` against a generated PingOne Terraform plan, with
     an interactive menu for reviewing resource counts and the target environment before applying.
3. **(Optional) Capture live client secrets** — if OAuth client secrets can't be exported directly
   from PingFederate, install the **Client Migration Tool** (see [downloads/](downloads/) below) on
   the PF host to capture `client_id`/`client_secret` pairs as they pass through the token/introspect/
   revoke/PAR endpoints, then import them into PingOne with
   [scripts/pingone-client-secret.sh](scripts/pingone-client-secret.sh) (or its Node equivalent,
   [scripts/pingone-client-secret.js](scripts/pingone-client-secret.js)) using `--action import` /
   `--action set-secret`, including CSV batch mode for bulk imports.

## Scripts

- **[cloud-migration-pingcli.sh](scripts/cloud-migration-pingcli.sh)** — pingcli-based apply/delete
  for a pingmigrate PingCLI export (resources, keys, applications, attribute mappings, resource
  grants). No state file; safe to re-run for idempotent resource types. Run with `--help` for full
  usage, interactive mode, and manifest details.
- **[cloud-migration-pingone.sh](scripts/cloud-migration-pingone.sh)** — Terraform-based apply/destroy
  against a PingOne environment, with an interactive menu (init/plan/apply/destroy, plus resource and
  environment detail views).
- **[pingone-client-secret.sh](scripts/pingone-client-secret.sh)** /
  **[pingone-client-secret.js](scripts/pingone-client-secret.js)** — import, set-secret, or rollback
  PingOne "imported" OIDC applications/custom resources with a custom `clientId`/`clientSecret`,
  singly or via CSV batch. Bash and Node implementations of the same behavior; requires `pingcli` and
  (for the bash version) `jq`.
- **[scripts/prod/cloud-migration-pingfederate-export.sh](scripts/prod/cloud-migration-pingfederate-export.sh)** —
  exports PingFederate OAuth clients, OIDC policies, access token managers, IdP/SP connections,
  adapters, auth policies, and signing keys (PKCS12, encrypted with a migration key) into a zip.
  Requires PingFederate 10.3+.
- **[scripts/prod/cloud-migration.sh](scripts/prod/cloud-migration.sh)** — earlier/simpler Terraform
  menu script (init/plan/apply/destroy) predating the pingcli-based flow.

## Downloads

- **[client-migration-tool-1.2.0-dist.zip](downloads/client-migration-tool-1.2.0-dist.zip)** — the
  Client Migration Tool distribution: a servlet filter
  (`com.ping.internal.ClientMigrationTool`) that PingFederate loads to capture `client_id`/
  `client_secret` pairs at the token/introspect/revoke/PAR endpoints, encrypting them with a
  `CLOUD_MIGRATION_KEY` you provide for later import into PingOne.
- **[client-migration-tool-INSTALL-SCRIPTED.md](downloads/client-migration-tool-INSTALL-SCRIPTED.md)** —
  automated install/uninstall via the bundled `install.sh` (Linux/macOS only).
- **[client-migration-tool-INSTALL-MANUAL.md](downloads/client-migration-tool-INSTALL-MANUAL.md)** —
  step-by-step manual install/uninstall, for hosts where the scripted installer can't run.

## Images

Screenshots and before/after diagrams of PingOne OIDC/SAML attribute mapping changes made during
migration, referenced from the pingmigrate documentation site.

## License

Scripts are distributed under the Apache License, Version 2.0 (see individual script headers).
