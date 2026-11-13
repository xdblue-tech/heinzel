---
name: arcane-cli
description: |
  Operate Arcane servers through arcane-cli. Use for containers, Compose
  projects, images, volumes, networks, environments, registries, GitOps,
  repositories, backups, variables, vulnerabilities, webhooks, activities,
  jobs, settings, authentication, and administration.

  Triggers: arcane-cli, Arcane server operations, Docker management,
  project up/down, container start/stop/redeploy, image updates,
  GitOps sync, environment management, and backups.
---

# Arcane CLI skill

## Before running commands

- Use the `arcane-cli` binary.
- Check `arcane-cli version` and the relevant command's `--help`. The installed binary may differ from the current repository source.
- The default configuration file is `~/.config/arcanecli.yml`. Use `--config <path>` to select another file.
- For host-specific server URLs and environment IDs, consult `~/.agentSecrets/arcane-cli/secrets.md` if present. Do not print credentials from it.
- Environment ID `"0"` identifies the manager's local Docker environment. Use the actual returned IDs for remote environments.
- Pass `--env <id>` explicitly for environment-scoped operations, except where a command-local flag shadows it.
- Do not infer missing permissions from an empty list alone. Check the server URL, authenticated identity, response status, filters, pagination, and environment access.
- Request only the permissions needed for the operation. Do not replace a restricted key with a full-admin key as a default troubleshooting step.

This reference follows the CLI source checked on 2026-09-06. Use installed command help to resolve version differences.

## Access and authorization

Never SSH into a server without explicit user permission. Use `arcane-cli` for remote container and project operations.

Only perform mutations within the user's requested scope. `--yes` skips CLI confirmation prompts; it does not supply user authorization.

Do not run cleanup, upgrades, restores, credential changes, or bulk operations merely to test connectivity.

## Configuration and authentication

```bash
# Inspect the config location and masked configuration
arcane-cli config path
arcane-cli config show

# Initialize a missing configuration file
arcane-cli config init

# Set configuration using key-value pairs
arcane-cli config set server-url https://arcane.example.com
arcane-cli config set api-key <api-key>
arcane-cli config set default-environment 0

# Multiple pairs are supported
arcane-cli config set server-url https://arcane.example.com default-environment 0

# Test connectivity
arcane-cli config test
arcane-cli auth me

# OIDC device authorization
arcane-cli auth oidc
arcane-cli auth login
arcane-cli auth refresh
arcane-cli auth logout

# Exchange a CI OIDC token
arcane-cli auth federated --help

# Manage the current user's API keys
arcane-cli auth keys list
arcane-cli auth keys create <name>
arcane-cli auth keys delete <id>

# Change the current user's password
arcane-cli auth password --help

# Select and persist the default environment interactively
arcane-cli environments switch
```

`config backup` moves the active configuration to a backup path. It is not a read-only inspection command.

Use an existing approved credential source. Do not include real API keys, passwords, JWTs, or refresh tokens in responses.

## Global flags and local overrides

| Flag | Purpose |
|---|---|
| `--config <path>`, `-c` | Select the configuration file |
| `--env <id>` | Override the configured environment |
| `--output text\|json` | Select output format |
| `--json` | Request JSON output |
| `--yes`, `-y` | Accept supported confirmation prompts |
| `--no-color` | Disable colored output |
| `--request-timeout <duration>` | Override the HTTP timeout |
| `--log-level <level>` | Set logging verbosity |
| `--log-json` | Format logs as JSON |

Command-local flags can override inherited flags:

- `containers create --env` sets container environment variables. It does not select the Docker environment.
- `templates download --output` selects an output file path.
- Some commands still print text or stream content even when a global JSON flag is present. Verify their output before scripting them.

For `containers create`, use a configuration whose `default_environment` already points to the intended host. Do not rely on moving `--env <id>` earlier in the command to bypass the flag conflict.

## Pagination

Most list commands support:

```bash
arcane-cli containers list --env 0 --limit 50 --start 0
arcane-cli containers list --env 0 --all
```

- `--all` returns every matching item, ignoring pagination.
- Do not combine `--all` with `--limit` or `--start`.
- `--start` is an offset. Database-backed lists round it down to a multiple of `--limit`.
- `containers list --all` does not mean "include stopped containers."
- `projects list --all` also includes archived projects.
- `templates list --all` has its own meaning: include remote templates.

Use command help for exceptions.

## JSON output

There is no universal JSON wrapper across CLI commands.

Paginated lists using the shared list implementation preserve the API response:

```json
{
  "success": true,
  "data": [],
  "pagination": {}
}
```

Many detail, count, and action commands print the decoded payload directly.

| Command | JSON structure |
|---|---|
| `containers list` | List envelope with `.data` |
| `projects list` | List envelope with `.data` |
| `containers get` | Container detail object directly |
| `projects get` | Project detail object directly |
| `networks get` | Network object directly |
| `volumes get` | Volume object directly |
| `containers counts` | Counts object directly |
| `projects counts` | Counts object directly |
| `images updates summary` | Summary object directly |
| `images updates check <image-ref>` | Update result directly |
| `images updates check --all` | Object keyed by image reference |
| `volumes counts` | Counts object directly |
| `networks counts` | Counts object directly |
| `jobs get` | Schedule configuration directly |
| `settings list` | Settings array directly |
| `updater status` | Status object directly |
| `system info` | Docker daemon information directly |

```bash
# Paginated list
arcane-cli containers list --env 0 --all --json \
  | jq '.data[] | {id, name: .names[0], state}'

# Detail response
arcane-cli projects get myproject --env 0 --json \
  | jq '{id, name, status}'

# Counts and summary responses
arcane-cli containers counts --env 0 --json \
  | jq '.runningContainers'

arcane-cli images updates summary --env 0 --json \
  | jq '.imagesWithUpdates'
```

Container list summaries and container detail responses have different schemas. Do not assume fields from one exist in the other.

Avoid printing whole detail responses when they can contain environment variables or credentials. Select only the fields needed for the task.

## Containers

```bash
arcane-cli containers list --env 0
arcane-cli containers list --env 0 --all
arcane-cli containers list --env 0 --updates has_update
arcane-cli containers list --env 0 --standalone true
arcane-cli containers list --env 0 --include-internal

arcane-cli containers get <id-or-name> --env 0
arcane-cli containers counts --env 0
arcane-cli containers updates --env 0 --all

arcane-cli containers start <id-or-name> --env 0
arcane-cli containers stop <id-or-name> --env 0
arcane-cli containers restart <id-or-name> --env 0
arcane-cli containers pause <id-or-name> --env 0
arcane-cli containers unpause <id-or-name> --env 0
arcane-cli containers kill <id-or-name> --env 0 --signal SIGTERM

# Run the container updater
arcane-cli containers update <id-or-name> --env 0

# Pull the image and recreate the container
arcane-cli containers redeploy <id-or-name> --env 0

# Apply an existing JSON edit payload and recreate the container
arcane-cli containers edit <id-or-name> --env 0 --file container-edit.json

# Configure automatic updates
arcane-cli containers autoupdate <id-or-name> --env 0 --enabled
arcane-cli containers autoupdate <id-or-name> --env 0 --enabled=false

# Create an image from a container
arcane-cli containers commit <id-or-name> --env 0 \
  --repository myapp --tag snapshot

arcane-cli containers delete <id-or-name> --env 0
```

`containers update` runs the updater. Use `containers edit --file` to change container configuration.

Without `--file`, `containers edit` prints the editable configuration, which can include secrets. Do not capture that output indiscriminately.

For creation, the selected config determines the target environment:

```bash
# arcane-prod.yml must already target the intended environment
arcane-cli --config /path/to/arcane-prod.yml containers create \
  --name myapp \
  --image nginx:latest \
  --port 8080:80 \
  --env APP_MODE=production \
  --volume /host/data:/data \
  --network shared-web \
  --restart unless-stopped

arcane-cli --config /path/to/arcane-prod.yml containers create \
  --file container.json
```

`containers delete --force` also permits removing a running container. It is not just a prompt override. `--volumes` removes associated anonymous volumes.

## Projects

```bash
arcane-cli projects list --env 0
arcane-cli projects list --env 0 --updates has_update
arcane-cli projects list --env 0 --archived true
arcane-cli projects updates --env 0 --all
arcane-cli projects get <id-or-name> --env 0
arcane-cli projects counts --env 0

arcane-cli projects up <id-or-name> --env 0
arcane-cli projects down <id-or-name> --env 0
arcane-cli projects restart <id-or-name> --env 0
arcane-cli projects pull <id-or-name> --env 0
arcane-cli projects redeploy <id-or-name> --env 0

# Pull and recreate selected services, or all services when omitted
arcane-cli projects upgrade <id-or-name> web worker --env 0

# Build services with Compose build directives
arcane-cli projects build <id-or-name> --env 0
arcane-cli projects build <id-or-name> --env 0 --service web

arcane-cli projects create --env 0 \
  --name myproject \
  --file compose.yml \
  --env-file .env

# Pass only the fields being changed
arcane-cli projects update <id-or-name> --env 0 --file compose.yml
arcane-cli projects update <id-or-name> --env 0 --name new-name
arcane-cli projects update <id-or-name> --env 0 --env-file .env

# Update an include file in the project workspace
arcane-cli projects workspace update <id-or-name> --env 0 --file override.yml

# Workspace access, subject to the secret-handling rules below
arcane-cli projects workspace cat <id-or-name> <path> --env 0
arcane-cli projects workspace download <id-or-name> <path> <output-file> --env 0

arcane-cli projects tags --env 0
arcane-cli projects tag <id-or-name> --env 0 --add production
arcane-cli projects tag <id-or-name> --env 0 --remove production
arcane-cli projects archive <id-or-name> --env 0
arcane-cli projects unarchive <id-or-name> --env 0
```

Project destruction removes the project and its containers. It also removes project files by default:

```bash
# Preserve project files
arcane-cli projects destroy <id-or-name> --env 0 --remove-files=false

# Default destruction also removes project files
arcane-cli projects destroy <id-or-name> --env 0

# Remove project volumes only when explicitly intended
arcane-cli projects destroy <id-or-name> --env 0 --remove-volumes
```

Do not use destruction when the request is only to stop project services.

## Images

```bash
arcane-cli images list --env 0
arcane-cli images list --env 0 --inuse
arcane-cli images list --env 0 --unused
arcane-cli images get <id-or-name> --env 0
arcane-cli images counts --env 0
arcane-cli images search nginx --env 0
arcane-cli images history <id-or-name> --env 0
arcane-cli images attestations <image> --env 0

arcane-cli images pull nginx:latest --env 0
arcane-cli images tag <id-or-name> myapp:backup --env 0
arcane-cli images export <id-or-name> image.tar --env 0
arcane-cli images upload image.tar --env 0
arcane-cli images remove <id-or-name> --env 0
arcane-cli images prune --env 0
arcane-cli images prune --env 0 --dangling

arcane-cli images build <context-dir-or-git-url> --help
arcane-cli images builds --env 0
arcane-cli images builds get <build-id> --env 0

# Check one or several image references
arcane-cli images updates check nginx:latest --env 0
arcane-cli images updates check nginx:latest redis:latest --env 0

# Check all images
arcane-cli images updates check --all --env 0
arcane-cli images updates summary --env 0
```

`images updates check` requires at least one image reference or `--all`. There is no implicit default image.

The older `check-all` and `check-image` commands remain hidden commands in the checked source. Prefer the visible `check` interface.

## Volumes

```bash
arcane-cli volumes list --env 0
arcane-cli volumes get <name> --env 0
arcane-cli volumes create --env 0 --name myvolume
arcane-cli volumes rename <name> <new-name> --env 0
arcane-cli volumes delete <name> --env 0
arcane-cli volumes prune --env 0
arcane-cli volumes counts --env 0
arcane-cli volumes sizes --env 0
arcane-cli volumes usage <name> --env 0

# Browse and transfer files
arcane-cli volumes workspace <name> --env 0
arcane-cli volumes workspace <name> <path> --env 0
arcane-cli volumes workspace cat <name> <path> --env 0
arcane-cli volumes workspace download <name> <path> <output-file> --env 0
arcane-cli volumes workspace put <name> <local-file> <path> --env 0

# Volume backups
arcane-cli volumes backups list <name> --env 0
arcane-cli volumes backups create <name> --env 0
arcane-cli volumes backups policy <name> --env 0
arcane-cli volumes backups policy update <name> --help
arcane-cli volumes backups restore <name> <backup-id> --env 0
arcane-cli volumes backups delete <backup-id> --env 0
arcane-cli volumes backups upload <backup-id> --help
arcane-cli volumes backups download <backup-id> <output-file> --env 0
arcane-cli volumes backups files <backup-id> --env 0
```

File reads remain subject to the secret-handling rules. Restores and workspace writes change data.

## Networks

```bash
arcane-cli networks list --env 0
arcane-cli networks get <id-or-name> --env 0
arcane-cli networks create --help
arcane-cli networks delete <id-or-name> --env 0
arcane-cli networks prune --env 0
arcane-cli networks counts --env 0
arcane-cli networks topology --env 0
arcane-cli networks connect <network> <container> --env 0
arcane-cli networks disconnect <network> <container> --env 0
```

## Environments

```bash
arcane-cli environments list --all
arcane-cli environments get <id>
arcane-cli environments test <id>
arcane-cli environments version <id>
arcane-cli environments switch
arcane-cli environments create --help
arcane-cli environments update <id> --help
arcane-cli environments delete <id>
```

An empty environment list is not proof of a missing permission. Diagnose the actual response and access scope.

Environment creation supports direct and edge configuration. Check `create --help` for pairing options. Generated pairing credentials may be shown once; do not expose them in conversation output.

## Registries and Git repositories

```bash
arcane-cli registries list
arcane-cli registries get <id>
arcane-cli registries create --help
arcane-cli registries update <id> --help
arcane-cli registries test <id>
arcane-cli registries usage
arcane-cli registries delete <id>

arcane-cli repos list
arcane-cli repos get <repository>
arcane-cli repos create --help
arcane-cli repos update <repository> --help
arcane-cli repos delete <repository>
arcane-cli repos test <repository>
arcane-cli repos branches <repository>
arcane-cli repos files <repository>
```

`registries sync` and `repos sync` are not registered commands in the checked source.

## GitOps

```bash
arcane-cli gitops list --env 0
arcane-cli gitops get <id-or-name> --env 0
arcane-cli gitops create --help
arcane-cli gitops update <id-or-name> --help
arcane-cli gitops delete <id-or-name> --env 0
arcane-cli gitops status <id-or-name> --env 0
arcane-cli gitops sync <id-or-name> --env 0
arcane-cli gitops files <id-or-name> --env 0
arcane-cli gitops import --help
```

A sync can change deployed project files and services. Run it only within the requested scope.

## System backups

Use top-level `backups` for system backups and `volumes backups` for volume backups.

```bash
arcane-cli backups list --env 0
arcane-cli backups create --env 0 --destination local
arcane-cli backups create --env 0 --s3-destination <destination>
arcane-cli backups restore <backup-id> --env 0
arcane-cli backups delete <backup-id> --env 0
arcane-cli backups upload <backup-id> --env 0 --s3-destination <destination>
arcane-cli backups discover --env 0 --s3-destination <destination>

arcane-cli backups policies --env 0
arcane-cli backups policies update --help
arcane-cli backups recovery generate --help
arcane-cli backups recovery set --help

arcane-cli backups s3 list
arcane-cli backups s3 get <destination>
arcane-cli backups s3 create --help
arcane-cli backups s3 update <destination> --help
arcane-cli backups s3 test <destination>
arcane-cli backups s3 delete <destination>
```

Recovery keys are secrets. Preserve the existing key when operating on encrypted backups.

`backups policies update --file` replaces the full policy collection. Inspect its help before using it for a partial change.

## Jobs and activities

```bash
# Schedule configuration
arcane-cli jobs get --env 0
arcane-cli jobs update --help

# Available jobs and execution
arcane-cli jobs list --env 0
arcane-cli jobs run <job-id> --env 0
arcane-cli jobs run <job-id> --env 0 --request-id <stable-request-id>

# Activity tracking
arcane-cli activities list --env 0
arcane-cli activities get <activity-id> --env 0
arcane-cli activities cancel <activity-id> --env 0
arcane-cli activities clear --env 0
```

Reuse the same `--request-id` when retrying an uncertain job submission.

A successful submission is not proof that the job completed. Check the returned activity or status before reporting completion.

## Updater and system

```bash
arcane-cli updater status --env 0
arcane-cli updater run --env 0
arcane-cli updater history --env 0 --limit 50

arcane-cli system info --env 0
arcane-cli system prune --env 0
arcane-cli system containers start --env 0
arcane-cli system containers start --env 0 --stopped
arcane-cli system containers stop --env 0
arcane-cli system convert "docker run --name web -p 8080:80 nginx:latest"

# Check or perform an Arcane server upgrade
arcane-cli system upgrade --env 0 --check
arcane-cli system upgrade --env 0

# Upgrade all eligible environments and inspect the latest job
arcane-cli system upgrade --all
arcane-cli system upgrade --all --status
```

Use the current command forms:

- `system info`
- `system containers start`
- `system containers stop`
- `system containers start --stopped`
- `system upgrade --check`

Do not use the old `docker-info`, `containers-start-all`, `containers-stop-all`, `start-stopped`, or `upgrade-check` command names.

`system upgrade` updates the Arcane server. `self-update` updates the local CLI binary.

## Settings and shared variables

```bash
arcane-cli settings list --env 0
arcane-cli settings public --env 0
arcane-cli settings update --help

arcane-cli variables list
arcane-cli variables create --help
arcane-cli variables update <id> --help
arcane-cli variables delete <id>
arcane-cli variables sync
arcane-cli variables sync --status
```

Variable creation and updates support `--key`, `--value`, `--secret`, `--all-environments`, and repeatable `--environment` flags.

Variable scope flags describe where the variable applies. They are separate from the global `--env` target.

## Vulnerabilities

```bash
arcane-cli vulnerabilities status --env 0
arcane-cli vulnerabilities summary --env 0
arcane-cli vulnerabilities list --env 0 --severity critical,high
arcane-cli vulnerabilities scan <image> --env 0
arcane-cli vulnerabilities image <image> --env 0
arcane-cli vulnerabilities image <image> --env 0 --summary
arcane-cli vulnerabilities ignored --env 0
arcane-cli vulnerabilities ignore <cve-id> --help
arcane-cli vulnerabilities unignore <ignore-id> --env 0
```

Ignoring a vulnerability changes its tracking state. Do not use it as a substitute for remediation.

## Webhooks

```bash
arcane-cli webhooks list --env 0
arcane-cli webhooks create --help
arcane-cli webhooks update <webhook-id> --env 0 --enabled=false
arcane-cli webhooks delete <webhook-id> --env 0
arcane-cli webhooks trigger <token>
```

Creation requires a name, target type, and action. Targets include containers, projects, the updater, and GitOps syncs. Use a compatible action and target ID.

Webhook tokens are credentials. Triggering a webhook performs its configured action.

## Administration

```bash
# API keys
arcane-cli admin keys list
arcane-cli admin keys create <name> --help
arcane-cli admin keys get <id>
arcane-cli admin keys update <id> --help
arcane-cli admin keys delete <id>

# Users
arcane-cli admin users list
arcane-cli admin users create --help
arcane-cli admin users get <user-id>
arcane-cli admin users update <user-id> --help
arcane-cli admin users delete <user-id>

# Roles and permissions
arcane-cli admin roles list
arcane-cli admin roles get <role-id>
arcane-cli admin roles create --help
arcane-cli admin roles update <role-id> --help
arcane-cli admin roles delete <role-id>
arcane-cli admin roles permissions
arcane-cli admin roles assignments <user-id>
arcane-cli admin roles assign <user-id> --help

# OIDC claim mappings
arcane-cli admin oidc list
arcane-cli admin oidc create --help
arcane-cli admin oidc update <mapping-id> --help
arcane-cli admin oidc delete <mapping-id>

# Federated credentials
arcane-cli admin federated list
arcane-cli admin federated get <credential-id>
arcane-cli admin federated create --help
arcane-cli admin federated update <credential-id> --help
arcane-cli admin federated delete <credential-id>

# Events
arcane-cli admin events list
arcane-cli admin events stats
arcane-cli admin events delete <event-id>

# Notifications
arcane-cli admin notifications settings get
arcane-cli admin notifications settings set --help
arcane-cli admin notifications settings delete <provider>
arcane-cli admin notifications test <provider>
```

`admin api-keys` remains an alias for `admin keys`. `admin oidc-mappings` remains an alias for `admin oidc`.

Role assignment replaces the user's assignments. Read existing assignments first and retain every assignment that should remain:

```bash
arcane-cli admin roles assign <user-id> \
  --role <role-id>:<environment-id> \
  --role <another-role-id>
```

## Compose templates

```bash
arcane-cli templates list
arcane-cli templates list --all
arcane-cli templates get <id-or-name>
arcane-cli templates content <template-id>
arcane-cli templates create --help
arcane-cli templates update <template-id> --help
arcane-cli templates delete <template-id>
arcane-cli templates download <template-id> --output compose.yml

arcane-cli templates default
arcane-cli templates default save --help

arcane-cli templates registries
arcane-cli templates registries update <registry-id> --help
arcane-cli templates registries delete <registry-id>
arcane-cli templates fetch --url <registry-url>
```

`templates variables` is not registered in the checked source. Registry deletion is `templates registries delete`, not `templates delete-registry`.

## Local utilities

```bash
arcane-cli doctor
arcane-cli version

arcane-cli generate secret --help
arcane-cli generate apikey --help
arcane-cli generate tls --help
arcane-cli generate mtls --help

arcane-cli self-update --help
arcane-cli self-update channel
arcane-cli self-update channel stable
arcane-cli self-update channel next

arcane-cli completion bash
arcane-cli completion zsh
arcane-cli completion fish
arcane-cli completion powershell
```

Secret and certificate generation can print credentials or write files. Use the requested output destination and do not echo generated secrets into the conversation.

## Configuration reference

Example `~/.config/arcanecli.yml`:

```yaml
server_url: https://arcane.example.com
api_key: ""
default_environment: "0"
log_level: info
federated_audience: ""
cli_update_channel: stable

pagination:
  default:
    limit: 20
  resources:
    containers:
      limit: 50
    images:
      limit: 100
    volumes:
      limit: 40
    networks:
      limit: 40
```

OIDC login stores `jwt_token` and `refresh_token` in the configuration.

Supported key-value settings include:

- `server-url`
- `api-key`
- `jwt-token`
- `environment` or `default-environment`
- `federated-audience`
- `log-level`
- `cli-update-channel`
- `default-limit`
- `resource-limit`

```bash
arcane-cli config set default-limit 25
arcane-cli config set resource-limit containers=50
arcane-cli config set resource-limit.images 100
```

Use `0` to clear a configured pagination limit.

## Project environment files and secrets

Project `.env` files can contain database passwords, API keys, and tokens.

- Do not read or print their values, even partially.
- Do not use `cat`, a file-reading tool, or workspace content commands to display them.
- Pass an approved file path through `--env-file` without exposing its contents.
- Inspect only valid assignment names when checking keys.
- Modify variables only when the user requested the change.
- Avoid commands that put secret values into logs or conversation output.
- Apply the same care to Compose files and container configuration when they contain inline secrets.

Use a parser that prints only assignment names. Broad patterns that match every non-comment line can expose multiline secret values.

```bash
# Print assignment keys only
awk '
  /^[[:space:]]*(export[[:space:]]+)?[A-Za-z_][A-Za-z0-9_]*[[:space:]]*=/ {
    key = $0
    sub(/^[[:space:]]*/, "", key)
    sub(/^export[[:space:]]+/, "", key)
    sub(/[[:space:]]*=.*/, "", key)
    print key
  }
' .env

# Check whether an assignment exists without printing its value
if rg -q '^[[:space:]]*(export[[:space:]]+)?DATABASE_URL[[:space:]]*=' .env; then
  echo present
else
  echo missing
fi

# Pass the file directly to the CLI
arcane-cli projects create --env 0 \
  --name myapp \
  --file compose.yml \
  --env-file .env
```

An assignment being present does not prove its value is nonempty or valid.

## Common pitfalls

- Check local flags before assuming global flags retain their meaning.
- `containers create --env` sets a container variable and shadows environment selection.
- Most list commands use `--all` to disable pagination.
- Most detail commands print direct JSON objects. Do not append `.data` automatically.
- Some commands accept a JSON flag but still emit text or stream content.
- `containers update` runs the updater; `containers edit --file` changes configuration.
- `projects workspace update` updates an include file.
- `projects destroy` removes project files by default.
- `images updates check` requires image references or `--all`.
- Empty lists and failed responses require diagnosis; they do not justify automatic privilege escalation.
- IDs and names are interchangeable only where the command supports resolution. Use exact IDs for ambiguous names and ID-only commands.
- After a mutation, verify the resulting resource or activity state before reporting success.
