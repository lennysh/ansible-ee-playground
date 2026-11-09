# Running EE builds from Ansible Automation Platform

This guide sets up a **"Build Execution Environment"** job template in
Automation Controller. The template runs [`build_ee.yml`](../build_ee.yml)
against the EE Builder VM, so anyone with access can build and publish an
EE by picking a file from a drop-down.

Everything in the main [README](../README.md) still applies: the EE files,
the naming rules and `config/settings.yml`. Only how the playbook is
launched and where the secrets come from is different.

- [How it works in AAP](#how-it-works-in-aap)
- [Prerequisites](#prerequisites)
- [Option A: configure AAP as code (recommended)](#option-a-configure-aap-as-code-recommended)
- [Option B: configure AAP in the UI](#option-b-configure-aap-in-the-ui)
- [Building an EE from AAP](#building-an-ee-from-aap)
- [Day-2 operations](#day-2-operations)
- [Troubleshooting](#troubleshooting)

---

## How it works in AAP

```text
 ┌──────────────────────── Ansible Automation Platform ────────────────────────┐
 │  Job Template "Build Execution Environment"                                 │
 │    project   : this git repo (synced on launch)                             │
 │    inventory : "EE Builders" → group ee_builders → ee-builder VM            │
 │    creds     : "EE Builder VM SSH" (Machine) + "EE Builder Secrets"         │
 │    survey    : which ee_definitions/ file? preview only? push?              │
 │                                                                             │
 │  Execution node runs build_ee.yml inside the job's execution environment    │
 └────────────────────────────────────┬────────────────────────────────────────┘
                                      │ SSH (as ee-builder user)
                                      ▼
                     ┌─────────── EE Builder VM ───────────┐
                     │ ~/ee-builds/<ee>_v<version>/        │
                     │ ansible-builder build (podman)      │
                     └───────┬────────────────────┬────────┘
            pull base image  │                    │  push to EACH EE's own destination:
                             ▼                    ▼  <destination>/<ee>:<version> + :latest
                    registry.redhat.io     quay.io/..., pah.example.com/..., ...
                                           (collections pulled from Private Automation Hub)
```

What changes compared with running on the CLI:

| | CLI on the builder VM | From AAP |
| :-- | :-- | :-- |
| Inventory | `inventory.yml` (localhost) | "EE Builders" inventory, host in group **`ee_builders`** |
| Builder collections | `ansible-galaxy collection install` | Installed automatically on project sync from `collections/requirements.yml` |
| Secrets | `config/vault.yml` | **"EE Builder Secrets"** credential. It injects the same `vault_registry_credentials` + `vault_hub_token` variables. |
| Destination | `destination:` in each EE file | Same. Nothing to configure in AAP per EE. |
| Picking the EE | `-e ee_file=...` | Survey drop-down |
| Reviewing a change | `-e render_only=true` | Survey **Preview only? = true**. The generated file is printed in the job output. |

## Prerequisites

### EE Builder VM

The VM setup from the main README ([One-time setup → step 1](../README.md#1-prepare-the-ee-builder-vm)), plus:

1. A dedicated user for builds that **AAP can SSH to**:

   ```bash
   sudo useradd ee-builder
   sudo loginctl enable-linger ee-builder     # stable rootless podman for non-interactive sessions
   sudo -u ee-builder mkdir -m 700 -p ~ee-builder/.ssh
   # add AAP's public key to ~ee-builder/.ssh/authorized_keys
   ```

2. Test it as that user: `podman info` and `ansible-builder --version`.
3. The user does **not** need sudo. Builds use rootless podman in its home
   directory (`~/ee-builds`, `~/.local/share/containers`).

### AAP

* **Organization Galaxy credentials** that can install `infra.ee_utilities`
  and `containers.podman`. AAP installs these on project sync. Either
  include galaxy.ansible.com, or sync both collections into your Private
  Automation Hub.
* Every collection used **inside** your EEs must be available in your
  Private Automation Hub. See the main README.
* An execution environment to **run** the job. The built-in
  *Default execution environment* works, because the build itself happens
  on the builder VM.
* Network: execution nodes → builder VM on SSH (22). Builder VM →
  registry.redhat.io, Private Automation Hub, destination registry.

### Repository

Commit your edited [`config/settings.yml`](../config/settings.yml)
(registry, hub host, AAP version) to the repo AAP will sync. **Don't** commit
`config/vault.yml`; in AAP the secrets come from a credential.

---

## Option A: configure AAP as code (recommended)

[`configure_aap.yml`](configure_aap.yml) uses the Red Hat CoP collection
[`infra.aap_configuration`](https://github.com/redhat-cop/infra.aap_configuration)
to create every object below. It's idempotent, so it's safe to re-run.

1. Install the collections. They're certified, so install from Automation
   Hub (configure the `[galaxy]` section in `ansible.cfg`):

   ```bash
   ansible-galaxy collection install -r aap/requirements.yml
   ```

2. Edit the values marked `CHANGE ME` in
   [`aap/vars/ee_builder_aap.yml`](vars/ee_builder_aap.yml): AAP URL, admin
   user, organization, git URL, builder VM hostname and user.
3. Add the AAP-only secrets to `config/vault.yml`: `vault_aap_password` and
   `vault_ee_builder_ssh_private_key` (see `config/vault.yml.example`).
4. Run it from any machine that can reach the AAP API:

   ```bash
   ansible-playbook aap/configure_aap.yml --ask-vault-pass
   ```

It creates:

| Type | Name | Purpose |
| :-- | :-- | :-- |
| Credential type | EE Builder Secrets | Registry credentials map (per host) + hub token, injected as extra vars |
| Credential | EE Builder Secrets | Values copied from `config/vault.yml` (`vault_registry_credentials`, `vault_hub_token`) |
| Credential | EE Builder VM SSH | Machine credential for the `ee-builder` user |
| Project | Ansible EE Playground | This repo, updated on every launch |
| Inventory | EE Builders | Builder VM host, in group `ee_builders` |
| Job template | Build Execution Environment | `build_ee.yml` with a survey |

---

## Option B: configure AAP in the UI

Create these in order. The names don't matter, but the **group name
`ee_builders`** and the **injector variable names** must match exactly.

### 1. Credential type: "EE Builder Secrets"

*Automation Execution → Infrastructure → Credential Types → Create*

Registry credentials are **one YAML field**, a map of registry host →
username/password, because each EE can push to a different registry. AAP
allows only one credential of each type on a job template, so a separate
credential per registry wouldn't work.

**Input configuration:**

```yaml
fields:
  - id: registry_credentials
    type: string
    label: Container registry credentials (YAML)
    multiline: true
    secret: true
    help_text: Map of registry host to username/password (pull base images, push EEs)
  - id: hub_token
    type: string
    label: Private Automation Hub API token
    secret: true
```

**Injector configuration:**

```yaml
extra_vars:
  vault_registry_credentials: "{{ registry_credentials }}"
  vault_hub_token: "{{ hub_token }}"
```

### 2. Credentials

* **EE Builder Secrets**: type *EE Builder Secrets*.
  * *Private Automation Hub API token*: the hub token.
  * *Container registry credentials (YAML)*: one entry for **every registry
    host** your EEs pull base images from or push to:

    ```yaml
    registry.redhat.io:
      username: "12345678|ee-builder"
      password: "..."
    quay.io:
      username: "myorg+ee_builder"
      password: "..."
    pah.example.com:
      username: "ee-builder"
      password: "..."
    ```

    Quote values that contain YAML special characters. The field is
    write-only after saving, so to change one password, paste the full map
    again. Option A regenerates it from `config/vault.yml`.
* **EE Builder VM SSH**: type *Machine*. Username `ee-builder`, plus the
  SSH private key whose public half is on the builder VM. No privilege
  escalation.
* *(Private repo only)* a *Source Control* credential for git.

### 3. Project: "Ansible EE Playground"

* Source control type: Git, URL of this repo, branch `main`
* Options: **Update revision on launch** ✔
* Wait for the first sync to succeed. That installs `infra.ee_utilities`
  and `containers.podman`.

### 4. Inventory: "EE Builders"

* Add a **host**: the builder VM's FQDN.
* Add a **group** named exactly **`ee_builders`** and put the host in it.

### 5. Job template: "Build Execution Environment"

| Field | Value |
| :-- | :-- |
| Inventory | EE Builders |
| Project | Ansible EE Playground |
| Playbook | `build_ee.yml` |
| Execution environment | Default execution environment |
| Credentials | EE Builder VM SSH **and** EE Builder Secrets |
| Concurrent jobs | Off (one build at a time on the VM) |

**Survey** (enable it after adding the questions):

| Question | Variable | Type | Choices / default |
| :-- | :-- | :-- | :-- |
| EE definition file | `ee_file` | Multiple choice, required | One line per file, e.g. `ee_definitions/ee-demo-minimal_v1.0.0.yml` |
| Preview only? | `render_only` | Multiple choice, required | `false` / `true`, default `false` |
| Push to registry? | `push` | Multiple choice, required | `true` / `false`, default `true` |

> If you'd rather not maintain the drop-down by hand, use a **Text**
> question for `ee_file` instead. Option A regenerates the list automatically.

---

## Building an EE from AAP

1. **Launch** "Build Execution Environment".
2. Pick the EE file. Leave *Preview only?* = `false` and *Push?* = `true`.
3. Follow the job output:
   * **Show build plan**: EE name, version tag, base image, destination
   * **Run the Ansible Builder program**: the build on the VM
   * **Push image to registry**: `:<version>` and `:latest`
4. To use the new EE, go to *Automation Execution → Infrastructure →
   Execution Environments → Create*:
   * Image: `<destination>/<ee-name>:<version>`, as shown in the job's
     *Show build plan* output. Pin the version tag
     rather than `latest` in production.
   * Registry credential: a *Container Registry* credential for your Private
     Automation Hub
   * Pull: *Only pull the image if not present before running*. The tag is
     immutable, so it never needs re-pulling.

**Reviewing a change before building:** launch with *Preview only?* =
`true`. The job prints the generated `execution-environment.yaml`, then
removes the build files from the VM.

---

## Day-2 operations

### Adding or releasing an EE

1. Add or rename the file in `ee_definitions/` in a merge request (see the
   main README) and merge it. If its `destination:` uses a registry host
   that's new to you, add that host to the *EE Builder Secrets* credential.
2. With Option A, **re-run `aap/configure_aap.yml`** so the new file appears
   in the survey drop-down. With Option B, edit the survey choices.
3. Launch the job template and pick the new file.

### Access control

* Grant the platform team **Admin** on the job template and project.
* Grant other teams **Execute** on the job template so they can build. They
  can't see the secrets or change what gets built, because that is
  controlled by the repo and merge-request reviews.

### Automating builds

* **Build on merge:** call the job template from your CI pipeline through the
  AAP API, or with `ansible.controller.job_launch`, passing `ee_file`
  as an extra var.
* **Patch base images:** to pick up CVE fixes, bump the PATCH version
  (`_v1.0.0` → `_v1.0.1`) and build. Each tag stays immutable.
* **Register the EE automatically:** add a second workflow step that runs
  `infra.aap_configuration` with a `controller_execution_environments`
  entry for the new image.

---

## Troubleshooting

| Symptom | Fix |
| :-- | :-- |
| `skipping: no hosts matched` | The builder VM isn't in a group named exactly `ee_builders` in the job's inventory. |
| `couldn't resolve module/action 'containers.podman...'` or role `infra.ee_utilities.ee_builder` not found | Project sync didn't install `collections/requirements.yml`. Check the organization's Galaxy credentials and the project sync output. |
| `Missing vault_hub_token` | Attach the **EE Builder Secrets** credential to the job template. |
| `authentication required` on push, or `push auth: none stored for <host>` in the build plan | Add that exact registry host (with port, if any) to the credential's registry YAML. |
| `vault_registry_credentials must map registry host -> {username, password}` | The registry YAML in the credential is malformed. Each host needs `username` and `password`. |
| `Permission denied (publickey)` | Check the Machine credential's user and key, and `~ee-builder/.ssh/authorized_keys` on the VM. |
| Podman warnings about `systemd user session` / cgroups | `sudo loginctl enable-linger ee-builder` on the VM. |
| `ansible-builder: command not found` | Install it on the **VM** (not in the job EE), in the `ee-builder` user's PATH. |
| The new EE file isn't in the drop-down | Re-run `aap/configure_aap.yml` (Option A) or edit the survey (Option B). |
| Builds fill the VM disk | Schedule `podman system prune -af` as the `ee-builder` user (removes ALL unused images). |
