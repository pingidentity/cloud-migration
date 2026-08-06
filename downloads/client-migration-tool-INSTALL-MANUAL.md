# Client Migration Tool — Manual Installation

This describes installing/uninstalling the tool manually, by hand. If you
are on a linux/mac platform running `/bin/sh`, you may use the script in
`bin/install.sh` which automates the install — see [client-migration-tool-INSTALL-SCRIPTED.md](https://pingidentity.github.io/cloud-migration/downloads/client-migration-tool-INSTALL-SCRIPTED.md)
for that flow. Use this guide only if you can't run that script (e.g.
restricted shell access, a host without `/bin/sh`, or you need to
review/apply each change by hand).

## What the tool does

![PingFederate Client Migration Tool (large)](https://pingidentity.github.io/cloud-migration/downloads/pingfederate-client-migration-tool.png)

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
- Scripted Installation Instructions (if you are on a non-linux/mac platform) - [client-migration-tool-INSTALL-SCRIPTED.md](https://pingidentity.github.io/cloud-migration/downloads/client-migration-tool-INSTALL-SCRIPTED.md)

## Prerequisites

- Shell access to the PingFederate host, with permission to stop/start the
  PingFederate service and modify files under its install directory.
- `awk`, `unzip`, and `zip` available on the host.
- The distribution package (`client-migration-tool-<version>-dist.zip`),
  containing:

  ```text
  client-migration-tool-<version>/
    install.sh                       (not needed for manual install)
    lib/
      client-migration-tool-<version>.jar
      _common.sh                     (not needed for manual install)
      client-test.sh                 (optional smoke test)
      run.conf
  ```

- Know your PingFederate root install directory — referred to below as
  `PF_ROOT_DIR` (the directory containing `bin/` and `server/default/deploy/`).

## 1. Extract the distribution package

Unzip the dist package to a working directory on the PingFederate host,
e.g. `/tmp/client-migration-tool/`. You only need the `lib/` contents:

```sh
unzip client-migration-tool-<version>-dist.zip -d /tmp/
cd /tmp/client-migration-tool-<version>
```

## 2. Back up the files you're about to change

Before modifying anything, copy the two files the tool touches so you can
restore them later if needed:

```sh
cp "$PF_ROOT_DIR/server/default/deploy/pf-runtime.war" /tmp/pf-runtime.war.bak
cp "$PF_ROOT_DIR/bin/run.conf" /tmp/run.conf.bak
```

(If `$PF_ROOT_DIR/bin/run.conf` doesn't exist yet, create an empty one first.)

## 3. Deploy the filter JAR

Copy the tool's JAR into PingFederate's deploy directory:

```sh
cp lib/client-migration-tool-<version>.jar "$PF_ROOT_DIR/server/default/deploy/"
```

## 4. Register the filter in `pf-runtime.war`

The filter must be added to `WEB-INF/web.xml` inside `pf-runtime.war`.

1. Extract just that file to a scratch directory:

   ```sh
   TMP_DIR=$(mktemp -d)
   cd "$TMP_DIR"
   unzip "$PF_ROOT_DIR/server/default/deploy/pf-runtime.war" WEB-INF/web.xml
   ```

2. Edit `WEB-INF/web.xml` in a text editor:
   - If a `<filter>` with `<filter-name>ClientMigrationToolFilter</filter-name>`
     already exists, skip this step — it's already installed.
   - Otherwise, add this block immediately after the **last** existing
     `</filter>` closing tag:

     ```xml
     <filter>
         <filter-name>ClientMigrationToolFilter</filter-name>
         <filter-class>com.ping.internal.ClientMigrationTool</filter-class>
     </filter>
     ```

   - Then, immediately after the **last** existing `</filter-mapping>`
     closing tag, add one mapping block per URL pattern (skip any that
     already exist):

     ```xml
     <filter-mapping>
         <filter-name>ClientMigrationToolFilter</filter-name>
         <url-pattern>/as/token.oauth2</url-pattern>
     </filter-mapping>
     <filter-mapping>
         <filter-name>ClientMigrationToolFilter</filter-name>
         <url-pattern>/as/introspect.oauth2</url-pattern>
     </filter-mapping>
     <filter-mapping>
         <filter-name>ClientMigrationToolFilter</filter-name>
         <url-pattern>/as/revoke_token.oauth2</url-pattern>
     </filter-mapping>
     <filter-mapping>
         <filter-name>ClientMigrationToolFilter</filter-name>
         <url-pattern>/as/par.oauth2</url-pattern>
     </filter-mapping>
     ```

3. Repack the updated file back into the WAR:

   ```sh
   zip "$PF_ROOT_DIR/server/default/deploy/pf-runtime.war" WEB-INF/web.xml
   rm -rf "$TMP_DIR"
   ```

## 5. Configure environment variables

The filter reads its configuration from environment variables at
PingFederate startup:

| Variable | Required | Description |
| - | - | - |
| `CLOUD_MIGRATION_KEY` | yes | Same key used by the PingFederate export script; encrypts captured credentials. Minimum 10 characters. |
| `CLIENT_MIGRATION_EXPORT_FILE` | yes | Path/file where exported client data is written. |
| `CLIENT_MIGRATION_LIMIT` | no (default `1000`) | Max number of unique client id/secret combinations collected before collection stops. |
| `CLIENT_MIGRATION_DUPLICATE_IDS` | no (default `false`) | If `true`, collect all id/secret combinations even when the same client ID reappears with a different secret. |
| `CLIENT_MIGRATION_INCLUDE_IDS` | no | Comma-separated client IDs to export exclusively. If unset, all clients are exported. |
| `CLIENT_MIGRATION_EXCLUDE_IDS` | no | Comma-separated client IDs to exclude from export. |

Create `$PF_ROOT_DIR/bin/client-migration-tool.conf` containing:

```sh
# Environment variables for Client Migration Tool
export CLOUD_MIGRATION_KEY=<your-key>
export CLIENT_MIGRATION_EXPORT_FILE=<path-to-export-file>
export CLIENT_MIGRATION_LIMIT=1000
export CLIENT_MIGRATION_DUPLICATE_IDS=false
export CLIENT_MIGRATION_INCLUDE_IDS=
export CLIENT_MIGRATION_EXCLUDE_IDS=
```

## 6. Wire the config into `run.conf`

`run.conf` needs to source that file at startup. If
`$PF_ROOT_DIR/bin/run.conf` doesn't already contain a
`# Client Migration Tool Configuration` section, append this:

```sh
# Client Migration Tool Configuration
CM_CONF="$PF_ROOT_DIR/bin/client-migration-tool.conf"
if [ -r "$CM_CONF" ]; then
  . "$CM_CONF"
fi
```

(This is the exact contents of `lib/run.conf` in the distribution package —
you can also just `cat lib/run.conf >> "$PF_ROOT_DIR/bin/run.conf"` after
adding the header comment line above it.)

## 7. Restart PingFederate

Restart the PingFederate service so it picks up the new `run.conf`
environment variables and redeploys `pf-runtime.war` with the filter.

## 8. Verify

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

## Uninstalling manually

1. Stop PingFederate.
2. Restore the backups taken in step 2:

   ```sh
   cp /tmp/pf-runtime.war.bak "$PF_ROOT_DIR/server/default/deploy/pf-runtime.war"
   cp /tmp/run.conf.bak "$PF_ROOT_DIR/bin/run.conf"
   ```

3. Remove the deployed JAR:

   ```sh
   rm -f "$PF_ROOT_DIR"/server/default/deploy/client-migration-tool-*.jar
   ```

4. Restart PingFederate.

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
`CLIENT_MIGRATION_EXCLUDE_IDS`/`CLIENT_MIGRATION_INCLUDE_IDS` values from
step 5. Each data line's third field is the client secret, encrypted with
`CLOUD_MIGRATION_KEY` — decrypt it with the same key during import into the
target environment.
