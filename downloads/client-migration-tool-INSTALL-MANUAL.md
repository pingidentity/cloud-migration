# Client Migration Tool — Automated Installation

This describes installing/uninstalling the tool using the interactive
`bin/install.sh` script, which automates the steps documented in
[INSTALL-MANUAL.md](INSTALL-MANUAL.md). Use that doc instead if you need to
apply the changes by hand or want to understand exactly what the script
does under the hood.

## What the tool does

The tool is a servlet filter (`com.ping.internal.ClientMigrationTool`,
filter name `ClientMigrationToolFilter`) that PingFederate loads inside
`pf-runtime.war`. It's mapped to four OAuth2 endpoints so it can observe
`client_id`/`client_secret` pairs as they pass through:

- `/as/token.oauth2`
- `/as/introspect.oauth2`
- `/as/revoke_token.oauth2`
- `/as/par.oauth2`

Captured credentials are encrypted with a key you provide
(`CLOUD_MIGRATION_KEY`) and written to an export file
(`CLIENT_MIGRATION_EXPORT_FILE`) for later import elsewhere.

## Downloads

- Current Distribution - [client-migration-tool-1.2.0-dist.zip](https://pingidentity.github.io/cloud-migration/downloads/client-migration-tool-1.2.0-dist.zip)

## Prerequisites

- **The script must be run on the PingFederate host itself, on Linux or
  macOS.** `install.sh` is a POSIX shell script (`#!/bin/sh -e`) and
  requires `/bin/sh` to be present — it is not supported on Windows.
  PingFederate should be stopped before you apply the filter (step 3 below)
  and restarted afterward to pick up the change.
- Shell access with permission to stop/start the PingFederate service and
  modify files under its install directory.
- `awk`, `unzip`, and `zip` available on the host — `install.sh` checks for
  `awk` and `unzip` up front and exits with an error if either is missing.
- The distribution package (`client-migration-tool-<version>-dist.zip`).
- Know your PingFederate root install directory — referred to below as
  `PF_ROOT_DIR` (the directory containing `bin/` and `server/default/deploy/`).

## 1. Extract the distribution package

Unzip the dist package to a working directory on the PingFederate host:

```sh
unzip client-migration-tool-<version>-dist.zip -d /tmp/
cd /tmp/client-migration-tool-<version>
```

This gives you:

```text
client-migration-tool-<version>/
  install.sh
  lib/
    client-migration-tool-<version>.jar
    _common.sh
    client-test.sh
    run.conf
```

## 2. Set `SERVER_ROOT_DIR`

`install.sh` needs to know where PingFederate is installed. Export it
before running the script:

```sh
export SERVER_ROOT_DIR=/path/to/pingfederate
```

If it's not set, the script prints a warning and most operations will fail
because they depend on locating `pf-runtime.war` and `run.conf` under that
directory.

## 3. Run the installer

```sh
sh install.sh
```

You'll see an interactive menu:

```text
##############################################################
#                     Client Migration Tool
#
#               Script Version: <version>
#        Server Root Directory: /path/to/pingfederate
#
#            CLOUD_MIGRATION_KEY: Not Set
#   CLIENT_MIGRATION_EXPORT_FILE: Not Set
#         CLIENT_MIGRATION_LIMIT: Not Set
# CLIENT_MIGRATION_DUPLICATE_IDS: Not Set
#   CLIENT_MIGRATION_INCLUDE_IDS: Not Set
#.  CLIENT_MIGRATION_EXCLUDE_IDS: Not Set
##############################################################
#  1. Set Environment Variables
#
#  2. Install Client Migration Tool Filter
#
#  3. UnInstall Client Migration Tool Filter
#
#  Q: Quit
##############################################################

Enter your choice:
```

### Option 1 — Set Environment Variables

Prompts for each value in order and writes them to
`$PF_ROOT_DIR/bin/client-migration-tool.conf`:

| Variable | Required | Description |
| - | - | - |
| `CLOUD_MIGRATION_KEY` | yes | Same key used by the PingFederate export script; encrypts captured credentials. Minimum 10 characters. |
| `CLIENT_MIGRATION_EXPORT_FILE` | yes | Path/file where exported client data is written. |
| `CLIENT_MIGRATION_LIMIT` | no (default `1000`) | Max number of unique client id/secret combinations collected before collection stops. |
| `CLIENT_MIGRATION_DUPLICATE_IDS` | no (default `false`) | If `true`, collect all id/secret combinations even when the same client ID reappears with a different secret. |
| `CLIENT_MIGRATION_INCLUDE_IDS` | no | Comma-separated client IDs to export exclusively. If unset, all clients are exported. |
| `CLIENT_MIGRATION_EXCLUDE_IDS` | no | Comma-separated client IDs to exclude from export. |

Run this before option 2 — the filter reads these values from
PingFederate's process environment at startup, and option 2 wires this
conf file into `run.conf` for you.

### Option 2 — Install Client Migration Tool Filter

- Backs up `pf-runtime.war` and `run.conf` to `./bak/` (next to
  `install.sh`), and writes before/after copies plus a log to
  `./audit/<timestamp>/`.
- Extracts `WEB-INF/web.xml` from `pf-runtime.war`, adds the
  `ClientMigrationToolFilter` `<filter>` definition and its four
  `<filter-mapping>` entries (skipping any that already exist), and
  repacks the WAR.
- Copies the tool's JAR into `$PF_ROOT_DIR/server/default/deploy/`.
- Appends a `# Client Migration Tool Configuration` block to
  `$PF_ROOT_DIR/bin/run.conf` that sources
  `client-migration-tool.conf` (skipped if that block already exists).

Safe to re-run — each sub-step checks whether it's already been applied
before making changes.

### Option 3 — UnInstall Client Migration Tool Filter

- Restores `pf-runtime.war` and `run.conf` from the backups in `./bak/`.
- Removes `client-migration-tool-*.jar` from
  `$PF_ROOT_DIR/server/default/deploy/`.

This only works if the backups from option 2 are still present in `./bak/`
relative to where you're running `install.sh` from.

### Q — Quit

Exits the script.

## 4. Restart PingFederate

Restart the PingFederate service so it picks up the `run.conf` changes and
redeploys `pf-runtime.war` with the filter installed.

## 5. Verify

- Check PingFederate's server log for filter initialization messages from
  `com.ping.internal.ClientMigrationTool` (it logs at startup, including an
  error if `CLOUD_MIGRATION_KEY` or `CLIENT_MIGRATION_EXPORT_FILE` are
  missing/invalid).
- Optionally run the included smoke test against a local PingFederate admin
  port (edit the `PF` URL at the top of the script first if it's not
  `https://localhost:9031`):

  ```sh
  sh lib/client-test.sh
  ```

- Confirm entries appear in the file configured as `CLIENT_MIGRATION_EXPORT_FILE`.

## Uninstalling

1. Stop PingFederate.
2. From the same directory where you originally ran `install.sh` (so the
   `./bak/` backups are found), run `sh install.sh` again and choose
   option **3**.
3. Restart PingFederate.

## Example export file output

Once installed and running, the file at `CLIENT_MIGRATION_EXPORT_FILE` will
look like this — a header describing the collection run, followed by one
encrypted `client_id`/`client_secret` line per captured credential:

```text
######################################################################################
# PingFederate Client Migration Tool

#
# This file contains proprietary encrypted security information and should not
# be shared with any other parties other than the Ping Identity Cloud Migration Tool.
#
# More information can be found at: https://library.pingidentity.com/page/collection-cm-help
#
#              Host/IP: 192.168.1.123
#              Version: 1.1.0
#   Started Collecting: 2026-07-18_13-16-41
#     Collection Limit: 1000
#
#  Included Client IDs:
#  Excluded Client IDs: [app-4, app-9]
#
######################################################################################
2025-07-18 13:16:41 | app-0 | 1vtKyBocflk2pl1unw9pGSuvkazrJtclbyVxfD54N+G/Vexzh…
2025-07-18 13:16:42 | app-1 | INw+F5lkvLSqmgGYz85s2ZsXHNNkcUgtPMjTPNUojICHN2rVQ…
```

The `Excluded Client IDs` (and `Included Client IDs`, if set) reflect the
`CLIENT_MIGRATION_EXCLUDE_IDS`/`CLIENT_MIGRATION_INCLUDE_IDS` values you set
in option 1. Each data line's third field is the client secret, encrypted
with `CLOUD_MIGRATION_KEY` — decrypt it with the same key during import into
the target environment.
