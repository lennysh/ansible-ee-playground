# Ansible EE Playground

Build and publish custom **Ansible Execution Environments (EEs)** from a
dedicated EE Builder VM, using the Red Hat Communities of Practice
[`infra.ee_utilities`](https://github.com/redhat-cop/ee_utilities) collection.

**One EE = one file.** Each file in [`ee_definitions/`](ee_definitions/)
fully describes one EE, including **where it's published**. The **filename
sets the image name and version**, and the file's **`destination:`** sets
the registry and namespace:

```text
ee_definitions/ansible-automation-platform-26/ee-casc-rhel9_v1.0.0.yml
               └─ folder (organising only) ─┘ └ee-name──┘ └version┘
                                                    │         │
   destination: quay.io/myorg/ansible-automation-platform-26  │
                │                                   │         │
                ▼                                   ▼         ▼
   quay.io/myorg/ansible-automation-platform-26/ee-casc-rhel9:1.0.0   (+ :latest)
```

Each EE can push to a different registry or namespace. Credentials are
picked per registry host automatically (see
[Destinations and registry credentials](#destinations-and-registry-credentials)).

You only edit YAML data; you don't write or maintain any build code.

There are two ways to run a build, both using the same files:

* **From Ansible Automation Platform** (typical): a *Build Execution
  Environment* job template with a survey runs the build on the EE Builder
  VM over SSH. See **[aap/README.md](aap/README.md)**.
* **From the CLI on the EE Builder VM**: handy for development and
  troubleshooting. Covered below.

> Want to see it in action? Follow [DEMO.md](DEMO.md).

---

## Table of contents

- [How it works](#how-it-works)
- [Running from AAP](aap/README.md)
- [Repository layout](#repository-layout)
- [Available EEs](#available-ees)
- [One-time setup](#one-time-setup)
- [Building an EE](#building-an-ee)
- [Adding a new EE](#adding-a-new-ee)
- [Releasing a new version of an EE](#releasing-a-new-version-of-an-ee)
- [Destinations and registry credentials](#destinations-and-registry-credentials)
- [Naming and versioning rules](#naming-and-versioning-rules)
- [Where do custom build commands go? (ansible-builder stages)](docs/ansible-builder-stages.md)
- [Why not just `podman build` a Containerfile? (measured comparison)](docs/podman-build-vs-ee-definition.md)
- [Troubleshooting](#troubleshooting)

---

## How it works

```text
 ee_definitions/<ee-name>_v<version>.yml      config/settings.yml + secrets
              │                                (config/vault.yml on the CLI,
              │                                 an AAP credential in AAP)
              └───────────────┬───────────────────────┘
                              ▼
                      build_ee.yml  (playbook, AAP job template or CLI)
                       • validates filename → name + tag
                       • loads the definition + global settings
                       • picks pull/push credentials by registry host
                       • runs on the EE Builder VM (inventory group ee_builders)
                              ▼
               infra.ee_utilities.ee_builder  (role)
                       • writes execution-environment.yaml + ansible.cfg
                       • pulls the base image (e.g. registry.redhat.io)
                       • ansible-builder build (podman)
                       • pushes <destination>/<name>:<version> and :latest
                              ▼
               The EE's destination registry (Private Automation Hub, Quay, ...)
                              ▼
               AAP Automation Controller uses the EE
```

* **Collections** are pulled from your **Private Automation Hub** during the
  build. The hub token is only present in the intermediate "galaxy" build
  stage. It is **not** in the final image.
* **Base images** come from wherever each EE's `base_image` points, usually
  `registry.redhat.io`.
* **Destination** is set **per EE** with `destination:`.
* The build runs on the **EE Builder VM** with rootless podman, in
  `~/ee-builds/<ee>_v<version>/`. That directory is removed afterwards
  unless you set `keep_context=true`.

## Repository layout

```text
.
├── build_ee.yml                  # The build playbook (AAP job template or CLI)
├── ee_definitions/               # ONE FILE PER EE  <-- you work here
│   ├── ee-demo-minimal_v1.0.0.yml
│   ├── ansible-automation-platform-26/     # sub-folders are just for organising
│   │   ├── ee-casc-rhel9_v1.0.0.yml
│   │   └── ...
│   └── ...                       # see "Available EEs"
├── templates/
│   └── ee-name_v0.1.0.yml        # Fully commented starting point for a new EE
├── config/
│   ├── settings.yml              # Global settings: hub, default base image, builder options
│   └── vault.yml.example         # Secrets template -> copy to vault.yml + encrypt
├── files/                        # Extra files to bake into EEs (CA certs, etc.)
├── docs/
│   ├── ansible-builder-stages.md # How build stages work + which hook to use
│   └── podman-build-vs-ee-definition.md  # Measured: hand-written Containerfile vs this approach
├── examples/
│   └── legacy-podman-build/      # A hand-maintained Containerfile + compare.sh
├── aap/                          # Running from AAP
│   ├── README.md                 #   setup guide (as code or via the UI)
│   ├── configure_aap.yml         #   creates credential type, creds, project, inventory, job template
│   ├── vars/ee_builder_aap.yml   #   ...the AAP object definitions
│   └── requirements.yml          #   infra.aap_configuration
├── collections/requirements.yml  # Collections build_ee.yml needs (AAP installs these on project sync)
├── ansible.cfg.example           # copy to ansible.cfg (git-ignored) for CLI use
└── inventory.yml                 # CLI use: localhost in group ee_builders
```

## Available EEs

| EE | Purpose | Destination | Collections from | Base image | Definition |
| :-- | :-- | :-- | :-- | :-- | :-- |
| `ee-casc-rhel8-git` | Config as Code for AAP 2.4 (collections from git) | `quay.io/lshirley/ansible-automation-platform-24` | Hub + git | `ansible-automation-platform/ee-minimal-rhel8:2.16` | [v1.0.0](ee_definitions/ansible-automation-platform-24/ee-casc-rhel8-git_v1.0.0.yml) |
| `ee-casc-rhel8` | Config as Code for AAP 2.4 | `quay.io/lshirley/ansible-automation-platform-24` | Hub | `ansible-automation-platform/ee-minimal-rhel8:2.16` | [v1.0.0](ee_definitions/ansible-automation-platform-24/ee-casc-rhel8_v1.0.0.yml) |
| `ee-casc-rhel9-git` | Config as Code for AAP 2.4 (collections from git) | `quay.io/lshirley/ansible-automation-platform-24` | Hub + git | `ansible-automation-platform/ee-minimal-rhel9:2.16` | [v1.0.0](ee_definitions/ansible-automation-platform-24/ee-casc-rhel9-git_v1.0.0.yml) |
| `ee-casc-rhel9` | Config as Code for AAP 2.4 | `quay.io/lshirley/ansible-automation-platform-24` | Hub | `ansible-automation-platform/ee-minimal-rhel9:2.16` | [v1.0.0](ee_definitions/ansible-automation-platform-24/ee-casc-rhel9_v1.0.0.yml) |
| `ee-casc-rhel8-git` | Config as Code for AAP 2.5 (collections from git) | `quay.io/lshirley/ansible-automation-platform-25` | Hub + git | `ansible-automation-platform/ee-minimal-rhel8:2.16` | [v1.0.0](ee_definitions/ansible-automation-platform-25/ee-casc-rhel8-git_v1.0.0.yml) |
| `ee-casc-rhel8` | Config as Code for AAP 2.5 | `quay.io/lshirley/ansible-automation-platform-25` | Hub | `ansible-automation-platform/ee-minimal-rhel8:2.16` | [v1.0.0](ee_definitions/ansible-automation-platform-25/ee-casc-rhel8_v1.0.0.yml) |
| `ee-casc-rhel9-git` | Config as Code for AAP 2.5 (collections from git) | `quay.io/lshirley/ansible-automation-platform-25` | Hub + git | `ansible-automation-platform/ee-minimal-rhel9:2.16` | [v1.0.0](ee_definitions/ansible-automation-platform-25/ee-casc-rhel9-git_v1.0.0.yml) |
| `ee-casc-rhel9` | Config as Code for AAP 2.5 | `quay.io/lshirley/ansible-automation-platform-25` | Hub | `ansible-automation-platform/ee-minimal-rhel9:2.16` | [v1.0.0](ee_definitions/ansible-automation-platform-25/ee-casc-rhel9_v1.0.0.yml) |
| `ee-aap-self-service-portal-rhel9` | AAP Self-Service Portal (AAP 2.6) | `quay.io/lshirley/ansible-automation-platform-26` | Hub + git | `ansible-automation-platform/ee-minimal-rhel9:2.16` | [v1.0.0](ee_definitions/ansible-automation-platform-26/ee-aap-self-service-portal-rhel9_v1.0.0.yml) |
| `ee-casc-rhel9-git` | Config as Code for AAP 2.6 (collections from git) | `quay.io/lshirley/ansible-automation-platform-26` | Hub + git | `ansible-automation-platform/ee-minimal-rhel9:2.16` | [v1.0.0](ee_definitions/ansible-automation-platform-26/ee-casc-rhel9-git_v1.0.0.yml) |
| `ee-casc-rhel9` | Config as Code for AAP 2.6 | `quay.io/lshirley/ansible-automation-platform-26` | Hub | `ansible-automation-platform/ee-minimal-rhel9:2.16` | [v1.0.0](ee_definitions/ansible-automation-platform-26/ee-casc-rhel9_v1.0.0.yml) |
| `ee-casc-rhel9-git` | Config as Code for AAP 2.7 (collections from git) | `quay.io/lshirley/ansible-automation-platform-27` | Hub + git | `ansible-automation-platform-27/ee-minimal-rhel9:2.16` | [v1.0.0](ee_definitions/ansible-automation-platform-27/ee-casc-rhel9-git_v1.0.0.yml) |
| `ee-casc-rhel9` | Config as Code for AAP 2.7 | `quay.io/lshirley/ansible-automation-platform-27` | Hub | `ansible-automation-platform/ee-minimal-rhel9:2.16` | [v1.0.0](ee_definitions/ansible-automation-platform-27/ee-casc-rhel9_v1.0.0.yml) |
| `ee-proxmox-rhel9` | Dynamic Proxmox inventory | `quay.io/lshirley/ansible-automation-platform` | Hub | `ansible-automation-platform/ee-minimal-rhel9:2.17` | [v1.0.0](ee_definitions/ansible-automation-platform/ee-proxmox-rhel9_v1.0.0.yml) |
| `ee-unifi-rhel9` | Dynamic UniFi inventory | `quay.io/lshirley/ansible-automation-platform` | git | `ansible-automation-platform/ee-minimal-rhel9:2.17` | [v1.0.0](ee_definitions/ansible-automation-platform/ee-unifi-rhel9_v1.0.0.yml) |
| `ee-demo-minimal` | Demo EE - network/data filters with ansible.utils | `pah.example.com/custom-ees` | Hub | `ansible-automation-platform/ee-minimal-rhel9:2.16` | [v1.0.0](ee_definitions/ee-demo-minimal_v1.0.0.yml) |
| `ee-openshift-rhel9` | Kubernetes and OpenShift automation | `pah.example.com/custom-ees` | Hub | `ansible-automation-platform/ee-minimal-rhel9:2.16` | [v1.0.0](ee_definitions/ee-openshift-rhel9_v1.0.0.yml) |
| `ee-xorriso-rhel9` | Building ISO images (xorriso) | `quay.io/lshirley` | - | `ubi9/ubi:latest` | [v1.0.0](ee_definitions/ee-xorriso-rhel9_v1.0.0.yml) |
| `ee-winrm-kerberos-rhel8` | Windows automation over WinRM with Kerberos authentication | `pah.example.com/custom-ees` | - | `ansible-automation-platform-24/ee-minimal-rhel8:1.0.0-965` | [v1.0.0](ee_definitions/examples/ee-winrm-kerberos-rhel8_v1.0.0.yml) |

*Hub* = your Private Automation Hub. *git* = public git repos: the EE Builder VM
(during the build) needs outbound HTTPS to github.com for these EEs.

> Keep this table up to date when you add or release EEs.

---

## One-time setup

> Running from AAP? Do steps 1 and 3 here, then follow [aap/README.md](aap/README.md).

### 1. Prepare the EE Builder VM

A RHEL 9 VM with:

* `podman`, `ansible-core` and `ansible-builder`, e.g. from the AAP repo:

  ```bash
  sudo subscription-manager repos --enable ansible-automation-platform-2.6-for-rhel-9-x86_64-rpms
  sudo dnf install -y podman ansible-core ansible-builder git
  ```

* A regular (non-root) user to run builds. Podman runs rootless.
* At least **40 GB free** in the user's home directory
  (`~/.local/share/containers`). EE builds use a lot of layer storage.
* Network access to `registry.redhat.io`, your Private Automation Hub, and
  your destination registry.

### 2. Clone this repo and install the builder collections

```bash
git clone <this-repo-url> ansible-ee-playground
cd ansible-ee-playground
cp ansible.cfg.example ansible.cfg      # local, git-ignored: edit freely
ansible-galaxy collection install -r collections/requirements.yml
```

Collections are installed into `./collections` (see `ansible.cfg`). To install
them from your Private Automation Hub instead of galaxy.ansible.com, edit
the `[galaxy]` section in your `ansible.cfg`. `ansible.cfg` is git-ignored,
so local settings such as `vault_password_file` or hub tokens are never
committed. AAP doesn't need it.

### 3. Edit global settings

Open [`config/settings.yml`](config/settings.yml) and set at least:

| Setting | What it is |
| :-- | :-- |
| `ee_ah_host` | Your Private Automation Hub (or AAP 2.5+ gateway) hostname, used to download collections |
| `ee_aap_version` | Your AAP version (`2.5`, `2.6`, ...) |
| `ee_default_destination` | *Optional.* Fallback destination for EE files without `destination:`. Leave unset to make every EE declare its own. |

Where images are pushed is **not** a global setting. Each EE file sets its own
`destination:`.

### 4. Create the secrets file

```bash
cp config/vault.yml.example config/vault.yml
vi config/vault.yml                 # one credentials entry per registry host + hub token
ansible-vault encrypt config/vault.yml
```

Add an entry under `vault_registry_credentials` for **every registry host**
your EEs pull base images from or push to. See
[Destinations and registry credentials](#destinations-and-registry-credentials).

`config/vault.yml` is git-ignored. Optionally save the vault password to a
`.vault_pass` file (also git-ignored) and uncomment `vault_password_file` in
`ansible.cfg` so you don't need `--ask-vault-pass`.

---

## Building an EE

```bash
ansible-playbook build_ee.yml --ask-vault-pass \
  -e ee_file=ee_definitions/ee-demo-minimal_v1.0.0.yml
```

Result: `<destination>/ee-demo-minimal:1.0.0` and `:latest` are pushed, where
`<destination>` is the `destination:` set in that file.

### Useful options

| Option | Effect |
| :-- | :-- |
| `-e render_only=true` | Only generate the ansible-builder `execution-environment.yaml` and print it. No build, no push, no vault needed. Good for reviewing changes. |
| `-e push=false` | Build the image locally but don't push it. Test it with `podman run`. |
| `-e keep_context=true` | Keep the build directory `~/ee-builds/<ee>_v<ver>/` on the builder VM for troubleshooting. |
| `-e ee_tag_latest=false` | Push only the version tag, without moving `latest`. |

To override a **role setting** from `config/settings.yml` on the command line,
use JSON syntax so booleans stay booleans. ansible-core 2.19+ rejects the
string `"false"` in conditionals:

```bash
-e '{"ee_update_base_images": false, "ee_verbosity": 3}'
```

### Testing a built EE locally

```bash
podman run --rm -it localhost/ee-demo-minimal:1.0.0 ansible-galaxy collection list
podman run --rm -it localhost/ee-demo-minimal:1.0.0 ansible --version
```

Or run a real playbook in it with
[ansible-navigator](https://ansible.readthedocs.io/projects/navigator/):

```bash
ansible-navigator run site.yml --eei localhost/ee-demo-minimal:1.0.0 --pp never -m stdout
```

---

## Adding a new EE

1. Copy the template and name the file after your EE. Use a sub-folder if
   it helps you organise:

   ```bash
   cp templates/ee-name_v0.1.0.yml ee_definitions/ansible-automation-platform-26/ee-network-rhel9_v1.0.0.yml
   ```

2. Set **`destination:`**. If it's a registry host you haven't used before,
   add its credentials (`vault_registry_credentials` on the CLI, the
   *EE Builder Secrets* credential in AAP).
3. Edit the rest: base image, collections, Python and system packages, and any
   custom build steps. The template documents every option. Before adding
   `build_steps`, read [docs/ansible-builder-stages.md](docs/ansible-builder-stages.md)
   to pick the right stage.
4. Check it renders: `ansible-playbook build_ee.yml -e ee_file=... -e render_only=true`
5. Build it without pushing and test it: `-e push=false`
6. Build and push, add it to the [Available EEs](#available-ees) table,
   open a merge request.

## Releasing a new version of an EE

The filename is the version, so a release means a new filename.

**Recommended: rename with `git mv`.** This keeps one current file per EE,
and git history shows exactly what changed between versions.

```bash
git mv ee_definitions/ee-demo-minimal_v1.0.0.yml ee_definitions/ee-demo-minimal_v1.1.0.yml
vi ee_definitions/ee-demo-minimal_v1.1.0.yml            # make your change
git commit -am "ee-demo-minimal 1.1.0: add ansible.posix"
git tag ee-demo-minimal/v1.1.0                          # optional, handy for rebuilds
ansible-playbook build_ee.yml --ask-vault-pass -e ee_file=ee_definitions/ee-demo-minimal_v1.1.0.yml
```

`git diff ee-demo-minimal/v1.0.0 ee-demo-minimal/v1.1.0` shows exactly what
changed between versions. To rebuild an old version, check out its tag.

**Alternative: keep versions side by side.** If you have to support two
versions at once, copy the file instead of renaming it. Both files can then
be built at any time.

## Destinations and registry credentials

### Where each EE is pushed

Every EE file says where it's published:

```yaml
destination: quay.io/myorg/ansible-automation-platform-26
```

* Format: `<registry-host>[:port]/<namespace>[/<more>/<levels>]`, all
  lowercase, no `https://`, and no image name or tag. Those come from the
  filename.
* The image is pushed as `<destination>/<ee-name>:<version>`, plus `:latest`.
* Different EEs can use different registries and namespaces. For example,
  multi-version EEs can go to `.../ansible-automation-platform` and
  2.6-only EEs to `.../ansible-automation-platform-26`.
* `config/settings.yml` can set `ee_default_destination` as a fallback. If
  neither is set, the build stops with a clear message, unless you build
  with `-e push=false`.

### How credentials are chosen

All registry credentials live in **one map keyed by registry host**:

```yaml
# CLI: config/vault.yml          AAP: "EE Builder Secrets" credential (same YAML)
vault_registry_credentials:
  registry.redhat.io:            # pull Red Hat base images
    username: "12345678|ee-builder"
    password: "..."
  quay.io:                       # push EEs whose destination is quay.io/...
    username: "myorg+ee_builder"
    password: "..."
  pah.example.com:               # push to (or pull mirrored base images from) your PAH
    username: "ee-builder"
    password: "..."
```

For each build, the playbook picks:

| Purpose | Credentials used |
| :-- | :-- |
| **Push** the EE | the entry for the host in the EE's `destination:` |
| **Pull** the base image | the entry for the host in the EE's `base_image` |

* The host must match **exactly**, including the port (`registry.example.com:5000`).
* **No entry for a host?** The build doesn't fail. It uses whatever
  `podman login` session the build user already has on the EE Builder VM,
  or pulls anonymously (fine for public images such as UBI). The build
  plan output always says which applies:

  ```text
  "Base image : registry.redhat.io/ansible-automation-platform/ee-minimal-rhel9:2.16",
  "  pull auth: stored credentials for registry.redhat.io",
  "Destination: quay.io/myorg/ansible-automation-platform-26/ee-casc-rhel9",
  "  push auth: stored credentials for quay.io",
  ```

* Credential values are never printed. The tasks that handle them use
  `no_log`, so they don't appear in AAP job output either.

### Recommendations

* Use **robot / service accounts** with push rights only to the namespaces
  they need: Quay robot accounts, a PAH user in a group with push rights,
  or a registry.redhat.io service account for pulls.
* If you prefer not to store push credentials at all, log in once on the
  builder VM as the build user (`podman login quay.io`) and leave that host
  out of `vault_registry_credentials`.

## Naming and versioning rules

The playbook enforces these rules and stops with a clear message if a file breaks them:

* Filename format: `<ee-name>_v<version>.yml`, in `ee_definitions/` or any
  sub-folder of it
  * `<ee-name>`: lowercase letters, digits and dashes, e.g. `ee-casc-rhel9`.
    Suggested pattern: `ee-<purpose>[-<platform>]-<os>`.
  * Sub-folders are only for organising. They don't affect the image name
    or destination. Mirroring the destination namespace, as this repo does,
    keeps things easy to find.
  * The same `<ee-name>` can appear in different folders, as long as the
    files push to different destinations.
  * `<version>`: [semantic version](https://semver.org) `MAJOR.MINOR.PATCH`,
    optionally with a suffix such as `1.2.0-rc1`.
* Never put `name:` or `tags:` inside the file. They come from the filename,
  so the file name and the image tag can't drift apart.
* Always set `destination:` (or `ee_default_destination` in settings).
* Pin your base image tag (`:2.16`, not `:latest`) and the important
  collection versions, so the same file always builds the same image.
* Suggested version bumps:
  * **PATCH**: rebuild for base-image CVE fixes or minor collection updates
  * **MINOR**: add collections or packages (backwards compatible)
  * **MAJOR**: remove content, change base image or ansible-core version

---

## Troubleshooting

| Symptom | Fix |
| :-- | :-- |
| `unauthorized` pulling the base image | Add or fix the entry for the base image's host (e.g. `registry.redhat.io`) in `vault_registry_credentials`. Test with `podman login <host>`. |
| `authentication required` / `unauthorized` on push | Add or fix the entry for the **destination's host** (exact host:port) in `vault_registry_credentials`, and check the account can push to that namespace. |
| `... has no "destination:"` | Add `destination:` to the EE file, set `ee_default_destination`, or build with `-e push=false`. |
| `destination '...' is not valid` | Use `host/namespace`, lowercase, no `https://`, no image name or tag. |
| Collection not found during build | The collection must be **synced into your Private Automation Hub** (published, rh-certified, validated or community repo). |
| `Conditionals must have a boolean result` | You passed a role boolean as `-e key=false`. Use JSON: `-e '{"key": false}'`. |
| A Python package fails to compile | Add the build tools under `dependencies.system`, e.g. `gcc`, `python3.11-devel`. |
| Need more build detail | `-e keep_context=true -e '{"ee_verbosity": 3}'`, then look in `~/ee-builds/` on the builder VM. |
| Out of disk space | `podman system prune -a` (removes ALL unused images). |
| `Destination directory .../files does not exist` | In `build_items` / `build_files`, list the bare filename (`corp-ca.pem`), not `files/corp-ca.pem`. |
| Custom `options:` broke the package manager | Setting `options:` replaces the defaults. Include `package_manager_path: /usr/bin/microdnf`. |

## References

* [infra.ee_utilities: ee_builder role](https://github.com/redhat-cop/ee_utilities/blob/devel/roles/ee_builder/README.md)
* [How ansible-builder stages work, and which hook to use](docs/ansible-builder-stages.md)
* [Hand-written Containerfile + podman build vs EE definitions: measured comparison](docs/podman-build-vs-ee-definition.md)
* [ansible-builder v3 definition format](https://ansible.readthedocs.io/projects/builder/en/stable/definition/)
* [Red Hat docs: Creating and using execution environments](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/)
