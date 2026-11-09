# How ansible-builder stages work (and where your commands belong)

Every EE definition in [`ee_definitions/`](../ee_definitions/) can add custom
Containerfile commands under `build_steps:`, in **8 hooks** across **4 build
stages**. Putting a command in the wrong hook is the most common cause of
EEs that are broken, bloated, or leak secrets. This guide explains what each
stage does, what survives into the image you ship, and how to choose the
right hook.

> Everything here was checked against **ansible-builder 3.1.1**: the
> Containerfile it generates, its helper scripts, and a test build. See
> [Verified behaviour](#verified-behaviour) at the end.

- [TL;DR](#tldr)
- [The big picture](#the-big-picture)
- [The generated Containerfile, annotated](#the-generated-containerfile-annotated)
- [Stage 1: base](#stage-1-base)
- [Stage 2: galaxy](#stage-2-galaxy)
- [Stage 3: builder](#stage-3-builder)
- [Stage 4: final](#stage-4-final)
- [Decision guide: "I want to..."](#decision-guide-i-want-to)
- [Rules of thumb](#rules-of-thumb)
- [Examples from this repo](#examples-from-this-repo)
- [See it for yourself](#see-it-for-yourself)
- [Verified behaviour](#verified-behaviour)

---

## TL;DR

| Hook | Runs in | Reaches the final image? | Typical use |
| :-- | :-- | :-- | :-- |
| `prepend_base` | base, **before** Python/Ansible setup | ✅ **Yes**: every stage inherits it | Repos, CA certs, proxy/index settings, Python swaps |
| `append_base` | base, **after** Python/Ansible setup | ✅ **Yes** | Fix-ups that need Python/Ansible present, in all stages |
| `prepend_galaxy` | galaxy, before collections install | ❌ No (except files under `/usr/share/ansible`) | `ansible.cfg` with hub token, git credentials for private collections |
| `append_galaxy` | galaxy, after collections install | ⚠️ Only changes under `/usr/share/ansible` | Install local collection tarballs, trim or patch collections |
| `prepend_builder` | builder, before wheels are built | ❌ No | Build-only toolchains (e.g. Rust) that bindep can't express |
| `append_builder` | builder, after wheels are built | ⚠️ Only changes under `/output` | Debugging the builder output |
| `prepend_final` | final, before collections/deps are installed | ✅ Yes | Rare: things the dependency install itself needs |
| `append_final` | final, after everything is installed | ✅ Yes | Runtime tools, config files, ENV, cleanup, verification |

**Rule of thumb:** use `append_final` for things you ship, `prepend_galaxy`
for secrets used only during the build, and `prepend_base` only when
*every* stage needs it.

In this repo the hooks go under `build_steps:` in an EE file. The
`infra.ee_utilities.ee_builder` role writes them out as
`additional_build_steps:` in the ansible-builder definition.

---

## The big picture

ansible-builder generates a **multi-stage Containerfile**. The four stages
share one parent, and the final image only receives specific directories
from the others:

```text
                     ┌───────────────────────────────┐
                     │ base_image (e.g. ee-minimal)  │
                     └───────────────┬───────────────┘
                                     │ FROM
                     ┌───────────────▼───────────────┐
                     │ 1. base                       │  prepend_base / append_base
                     │  pip, (python, ansible-core,  │
                     │  ansible-runner if requested) │
                     └──┬────────────┬────────────┬──┘
           FROM base    │            │ FROM base  │  FROM base
     ┌──────────────────▼───┐  ┌─────▼──────────┐ │
     │ 2. galaxy            │  │ 3. builder     │ │
     │ install collections  │  │ build Python   │ │
     │ & roles              │  │ wheels, work   │ │
     │ → /usr/share/ansible │  │ out RPM list   │ │
     └──────────┬───────────┘  │ → /output      │ │
                │              └───────┬────────┘ │
                │  COPY /usr/share/    │ COPY     │
                │  ansible             │ /output  │
                │       (also copied into builder)│
                │              ┌───────▼──────────▼───┐
                └─────────────►│ 4. final             │  prepend_final / append_final
                               │ install RPMs + wheels│
                               │ = THE IMAGE YOU SHIP │
                               └──────────────────────┘
```

Three things follow from this layout:

1. **Base is inherited by everything.** Anything done in `prepend_base` or
   `append_base` (files, packages, `ENV`, repo configs) is present in galaxy,
   builder **and the final image**.
2. **Galaxy and builder are temporary.** Their layers are thrown away. Only
   `/usr/share/ansible` (from galaxy) and `/output` (from builder) are copied
   forward. A file you create elsewhere in those stages, or an `ENV` you set
   there, never reaches the final image.
3. **Final starts again from base**, not from builder. Compilers or headers
   installed in builder don't come along. Only the finished wheels and the
   list of runtime RPMs do.

---

## The generated Containerfile, annotated

This is what ansible-builder 3.1.1 generates for a small EE with a
`HOOK-...` marker in every hook (trimmed of repeated `ARG` lines). To see
the one for your own EE, see [See it for yourself](#see-it-for-yourself).

```dockerfile
# ───────────────────────── 1. BASE ─────────────────────────
FROM $EE_BASE_IMAGE AS base
USER root
ENV PIP_BREAK_SYSTEM_PACKAGES=1
COPY _build/scripts/ /output/scripts/                 # helper scripts used by later stages
COPY _build/scripts/entrypoint /opt/builder/bin/entrypoint
RUN echo HOOK-prepend_base                            # ◀── prepend_base
#   only if dependencies.python_interpreter.package_system is set:
#   RUN $PKGMGR install $PYPKG -y
RUN /output/scripts/pip_install $PYCMD                # makes sure pip exists (ensurepip)
#   only if dependencies.ansible_core / ansible_runner are set:
#   RUN $PYCMD -m pip install --no-cache-dir $ANSIBLE_INSTALL_REFS
RUN echo HOOK-append_base                             # ◀── append_base

# ───────────────────────── 2. GALAXY ───────────────────────
FROM base AS galaxy
RUN echo HOOK-prepend_galaxy                          # ◀── prepend_galaxy
RUN /output/scripts/check_galaxy                      # fails if ansible-galaxy doesn't work
COPY _build /build                                    # NOTE: build files arrive AFTER prepend_galaxy
WORKDIR /build
RUN ansible-galaxy role install       -r requirements.yml --roles-path       /usr/share/ansible/roles
RUN ansible-galaxy collection install -r requirements.yml --collections-path /usr/share/ansible/collections
RUN echo HOOK-append_galaxy                           # ◀── append_galaxy

# ───────────────────────── 3. BUILDER ──────────────────────
FROM base AS builder
WORKDIR /build
RUN $PYCMD -m pip install --no-cache-dir bindep pyyaml packaging
RUN echo HOOK-prepend_builder                         # ◀── prepend_builder
COPY --from=galaxy /usr/share/ansible /usr/share/ansible
COPY _build/requirements.txt requirements.txt         # your dependencies.python
COPY _build/bindep.txt bindep.txt                     # your dependencies.system
RUN $PYCMD /output/scripts/introspect.py introspect ...   # merge yours + every collection's requirements
RUN /output/scripts/assemble                          # install [compile] RPMs, build wheels → /output
RUN echo HOOK-append_builder                          # ◀── append_builder

# ───────────────────────── 4. FINAL ────────────────────────
FROM base AS final
RUN echo HOOK-prepend_final                           # ◀── prepend_final
RUN /output/scripts/check_ansible $PYCMD              # fails if ansible / ansible-runner are missing
COPY --from=galaxy /usr/share/ansible /usr/share/ansible
COPY --from=builder /output/ /output/
RUN /output/scripts/install-from-bindep && rm -rf /output/wheels   # runtime RPMs + pip from wheels
RUN chmod ug+rw /etc/passwd
RUN mkdir -p /runner && chgrp 0 /runner && chmod -R ug+rwx /runner
WORKDIR /runner
RUN $PYCMD -m pip install --no-cache-dir 'dumb-init==1.2.5'
RUN echo HOOK-append_final                            # ◀── append_final
RUN rm -rf /output
LABEL ansible-execution-environment=true
USER 1000
ENTRYPOINT ["/opt/builder/bin/entrypoint", "dumb-init"]
CMD ["bash"]
```

---

## Stage 1: base

**Purpose:** prepare the common foundation that every other stage, and the
final image, is built `FROM`.

**What ansible-builder does here:**

1. Copies its helper scripts to `/output/scripts/`, plus the container
   entrypoint.
2. If `dependencies.python_interpreter.package_system` is set: installs that
   Python with the package manager.
3. Makes sure `pip` exists for the chosen Python (`ensurepip`).
4. If `dependencies.ansible_core` / `ansible_runner` are set: `pip install`s
   them. They're **not** needed with Red Hat `ee-minimal` / `ee-supported`
   images, which already include both.

**What reaches the final image:** everything, because final is `FROM base`.

### `prepend_base`

Runs **before** Python, pip and Ansible are installed. Use it for things the
rest of the build depends on, in *every* stage:

* **Corporate CA certificates**: galaxy needs them to reach your hub over
  TLS, builder needs them for pip, and final needs them at runtime.
* **Extra package repositories** (EPEL, internal mirrors) via repo files
  or release RPMs.
* **Proxy settings** (`ENV HTTPS_PROXY=...`) and a **private PyPI index**
  (`ENV PIP_INDEX_URL=...`). pip runs in builder *and* final, and an `ENV`
  only reaches both if it's set in base.
* **Switching the system Python** (see the [unifi example](#ee-unifi-rhel9-python-swap-in-prepend_base)),
  because every later stage must use the same interpreter.

### `append_base`

Runs **after** Python and Ansible are installed. Use it for fix-ups that need
them present and should apply to every stage, e.g. pinning or upgrading
`pip`/`setuptools`, or checking the `ansible-core` version early so a bad
base image fails fast.

### Cautions

* **Base is the most expensive place for anything.** It's in every stage,
  it's shipped, and changing it invalidates the build cache for all later
  steps.
* **Never put secrets in base.** Base layers are part of the final image.
  An `ENV TOKEN=...` or a copied credentials file in base ships to every
  user of the EE.
* If something is only needed at runtime, `append_final` is cheaper and
  just as effective.

---

## Stage 2: galaxy

**Purpose:** install the Ansible **collections and roles** from
`dependencies.galaxy` into `/usr/share/ansible`.

**What ansible-builder does here:**

1. Runs `check_galaxy`. The build fails if `ansible-galaxy` doesn't work
   (e.g. no ansible-core in the base image).
2. Copies the whole `_build/` context directory to `/build`. This
   includes your `additional_build_files`, which this repo calls `build_files`.
3. `ansible-galaxy role install` and `ansible-galaxy collection install`
   into `/usr/share/ansible/{roles,collections}`.

**What reaches the final image:** **only `/usr/share/ansible`.** Files
elsewhere, packages and `ENV` set in this stage are discarded.

### `prepend_galaxy`

Runs before collections are installed. This is the **right place for
secrets needed only while downloading content**, because nothing outside
`/usr/share/ansible` ships:

* **`ansible.cfg` pointing at Private Automation Hub, with a token.** The
  `infra.ee_utilities.ee_builder` role does this for you: with
  `ee_pull_collections_from_hub: true` it adds
  `ADD _build/configs/ansible.cfg /etc/ansible/ansible.cfg` to
  `prepend_galaxy`. That's why the hub token never reaches the final image.
* **Credentials for private git-hosted collections**: `~/.gitconfig`,
  `~/.git-credentials`, SSH keys and `known_hosts`.
* `ENV ANSIBLE_GALAXY_*` settings that only affect installation.

> ⚠️ **`/build` doesn't exist yet in `prepend_galaxy`.** `COPY _build /build`
> runs *after* this hook. Reference build files from the build context
> instead: `ADD _build/configs/x /dest` or `COPY _build/configs/x /dest`.
>
> ⚠️ If your EE defines its own `prepend_galaxy`, the ee_builder role appends
> its `ansible.cfg` ADD **after** your commands, so `ansible.cfg` isn't in
> place yet while your commands run.

### `append_galaxy`

Runs after collections are installed, with all build files under `/build`.
Use it to change what's in `/usr/share/ansible` before it's copied on:

* **Install a local collection tarball** you shipped via `build_files`:
  `RUN ansible-galaxy collection install /build/collections/my_ns-my_coll-1.0.0.tar.gz -p /usr/share/ansible/collections`
* **Trim collections** to shrink the image, e.g. delete `tests/`
  directories under `/usr/share/ansible/collections`.
* **Patch a collection** (a temporary hotfix). Document why, and remove it
  when upstream is fixed.

### Cautions

* `pip install` or `microdnf install` here is **wasted**: those changes
  don't reach the final image. Use `dependencies.python` / `dependencies.system`
  or final-stage hooks.
* Intermediate layers aren't in the image you ship, but they **do** stay in
  the builder VM's podman cache until pruned. Treat the builder VM as
  sensitive. This repo prunes dangling images after each build
  (`ee_prune_images: true`).

---

## Stage 3: builder

**Purpose:** work out every Python and system dependency, and **compile
Python packages into wheels** so the final image doesn't need compilers.

**What ansible-builder does here:**

1. Installs its own tools (`bindep`, `pyyaml`, `packaging`). They're used
   only in this stage.
2. Copies in the collections from galaxy.
3. `introspect`: merges **your** `dependencies.python` / `dependencies.system`
   with the `requirements.txt` / `bindep.txt` **declared inside every
   installed collection**. That's how a collection's own Python needs get
   installed without you listing them. For example, `community.proxmox`
   brings in `proxmoxer`.
4. `assemble`:
   * installs system packages tagged with the bindep **`compile`** profile
     (compilers, `-devel` headers), **here only**;
   * builds or downloads wheels for all Python requirements into
     `/output/wheels`;
   * writes the list of **runtime** system packages to
     `/output/bindep/run.txt`.

**What reaches the final image:** only `/output`: wheels, the runtime RPM
list and the requirements files. Final installs from them, then deletes
`/output`.

### `prepend_builder`

Runs before wheels are built. Use it for **build-only tooling that bindep
can't install**, for example a Rust toolchain via `curl` for packages
without a prebuilt wheel, or a compiler from a non-default repo. None of
it ships.

### `append_builder`

Runs after wheels are built. Mostly useful for **debugging**:
`RUN ls -R /output && cat /output/bindep/run.txt` shows exactly what the
final stage will install.

### The `[compile]` profile: keep build tools out of your image

bindep lines without a profile are **runtime** dependencies. They're
installed in builder **and** in final. Tag build-only packages with
`compile` so they stay in builder:

```yaml
dependencies:
  system:
    - jq [platform:rpm]                      # runtime  → in the final image
    - gcc [platform:rpm compile]             # build-only → builder stage only
    - python3.11-devel [platform:rpm compile]
```

In a test build, adding an **untagged** `gcc` made the final image
**138 MB larger** (430 MB → 568 MB) and left a compiler in it. With
`[compile]`, `gcc` was absent from the final image.

> Only tag a package `compile` if nothing needs it at **runtime**. Headers
> (`-devel`), `gcc`, `make` and `pkg-config` usually qualify. The shared
> libraries a compiled module links against usually don't, but RHEL
> normally pulls those in as dependencies of the runtime packages.

---

## Stage 4: final

**Purpose:** produce the image you ship. It starts again `FROM base`, takes
the finished artifacts from galaxy and builder, and installs them.

**What ansible-builder does here:**

1. `check_ansible`: fails the build if `ansible`, `ansible-runner`, or their
   Python imports are missing.
2. Copies `/usr/share/ansible` from galaxy and `/output` from builder.
3. `install-from-bindep`: installs the runtime system packages from
   `run.txt`, then pip-installs the Python requirements from the prebuilt
   wheels. No compiler needed.
4. Prepares OpenShift-friendly permissions (`/etc/passwd`, `/runner` group
   0), installs `dumb-init`, removes `/output`.
5. Switches to `USER 1000` and sets the entrypoint.

**What reaches the final image:** everything in this stage.

### `prepend_final`

Runs **before** `check_ansible` and before collections and dependencies are
installed. Needed less often than you'd think. Use it only for things the
dependency install itself relies on in this stage, e.g. a repo file needed
by a runtime RPM from `run.txt` that isn't already set up in base. If every
stage needs it, use `prepend_base` instead.

### `append_final`

Runs after **everything** is installed, still as **root**. This is the
default place for anything that should be in the image you ship:

* **Runtime CLI tools** that aren't RPMs or Python packages (`yq`,
  `kubectl`, `oc`).
* **Configuration files** in `/etc`, and **runtime `ENV`** (`ANSIBLE_*`
  defaults, `TZ`).
* **Restoring things the minimal base strips out.** Check first that it's
  really missing: older ee-minimal images shipped without zoneinfo data,
  and many EEs added `microdnf reinstall tzdata -y`. Current images include
  it, and that command now **fails the build** with "The transaction was
  empty". See the [casc example](#ee-casc--compile-tools-matching-python-headers-and-a-stale-tzdata-step).
* **Cleanup**: `microdnf clean all`, removing caches. Clean up in the same
  stage that made the mess; cleaning in final can't shrink layers from
  other stages.
* **Verification**, so a broken image fails the build instead of a job
  later: `RUN ansible-galaxy collection list`,
  `RUN python3 -c "import netaddr"`, `RUN jq --version`.

### Cautions

* Hooks run as **root**, but the image **runs as UID 1000**. On OpenShift
  it runs as a random UID with group 0. Files that must be writable at
  runtime need group 0 and group permissions:
  `RUN chgrp -R 0 /path && chmod -R g+rwX /path`.
* `ENV` set here **does** apply at runtime.

---

## Decision guide: "I want to..."

| I want to... | Put it in | Why |
| :-- | :-- | :-- |
| Add a collection / role | `dependencies.galaxy` | Declarative. Installed in galaxy. |
| Add a Python package | `dependencies.python` | Built as a wheel in builder, installed in final. |
| Add an RPM needed at runtime | `dependencies.system` (untagged) | Installed in final from `run.txt`. |
| Add an RPM needed only to compile | `dependencies.system` with `[compile]` | Stays out of the final image. |
| Trust a corporate CA | `prepend_base` | Every stage makes TLS connections (hub, PyPI, runtime). |
| Use EPEL / an internal RPM repo | `prepend_base` | Needed wherever packages are installed. |
| Use a proxy or private PyPI index | `prepend_base` (`ENV ...`) | pip runs in builder **and** final. ENV only crosses stages from base. |
| Pull collections from Private Automation Hub | `prepend_galaxy` (the role does it) | Token is used during install and isn't shipped. |
| Pull collections from a private git repo | `prepend_galaxy` | Credentials don't reach the final image. |
| Install a collection from a local tarball | `build_files` + `append_galaxy` | `/build` exists there, and the result lands in `/usr/share/ansible`. |
| Remove bulk from installed collections | `append_galaxy` | Change it before it's copied to final. |
| Install a Rust/C toolchain for a build | `prepend_builder` | Build-only, not shipped. |
| Debug which deps were detected | `append_builder` | `/output` is fully populated there. |
| Add a CLI binary (yq, oc, kubectl) | `append_final` | Runtime only. Cheapest place. |
| Set runtime environment variables | `append_final` (`ENV ...`) | Applies when the EE runs. |
| Replace the system Python | `prepend_base` | Every stage must use the same interpreter. |
| Fail the build if something's missing | `append_final` | Tests the image exactly as shipped. |
| Clean package caches | `append_final` (and any stage you installed in) | Each stage is cleaned on its own. |

---

## Rules of thumb

1. **Use declarative dependencies before `RUN` commands.** `dependencies.galaxy`,
   `python` and `system` get dependency resolution, collection
   introspection, wheel building and the runtime/compile split for free.
2. **Only base `ENV` is inherited by later stages.** An `ENV` in galaxy or
   builder doesn't reach final, and an `ENV` in final only applies to final.
3. **Files flow forward only through `/usr/share/ansible` (galaxy) and
   `/output` (builder).** Everything else in those stages is thrown away.
4. **Secrets belong in galaxy or builder, never in base or final.**
5. **`_build/` is the build context**, so you can `COPY _build/...` /
   `ADD _build/...` in any stage. The `/build` *directory* only exists in
   galaxy, after `COPY _build /build`, and builder's `/build` only contains
   `requirements.txt` and `bindep.txt`.
6. **Tag compile-only system packages `[compile]`.**
7. **Know your package manager.** Red Hat EE images use `microdnf`. A plain
   UBI image uses `dnf` (see `ee-xorriso-rhel9`). Use the same tool in your
   `RUN` commands.
8. **Order inside a hook matters.** The commands in one hook run top to
   bottom as consecutive layers.

---

## Examples from this repo

### `ee-casc-*`: compile tools, matching Python headers and a stale tzdata step

```yaml
dependencies:
  system:
    - python3.12-devel [platform:rpm compile]   # v1.0.0 had python3.11-devel, untagged
    - pkg-config [platform:rpm compile]
    - systemd-devel [platform:rpm compile]
    - gcc [platform:rpm compile]
build_steps:
  append_final:
    # v1.0.0 had "- RUN microdnf reinstall tzdata -y" here (removed, see below)
    - RUN microdnf clean all
```
* The four system packages are only needed to **compile** C extensions
  that the collections need, such as `systemd-python` from `ansible.eda`.
  In `v1.0.1` they're tagged `compile`. A test build of the same dependency
  set (`ansible.eda` + these packages) on `ee-minimal-rhel9:2.16` gave:

  | Variant | Result |
  | :-- | :-- |
  | `python3.11-devel`, untagged (as in `v1.0.0`) | ❌ build fails: `Python.h: No such file or directory` |
  | `python3.12-devel`, untagged | ✅ 697 MB, ships gcc and systemd-devel |
  | `python3.12-devel`, `[compile]` (`v1.0.1`) | ✅ **495 MB (−202 MB, −29%)**, no gcc. `import systemd.journal` still works. |

* **The `-devel` package must match the base image's Python.** The `:2.16`
  images are rebuilt over time and now ship **Python 3.12**, so the
  `python3.11-devel` in `v1.0.0` stopped working. Check with
  `podman run --rm --entrypoint python3 <base_image> --version` whenever
  you change or refresh a base image.
* **Steps that worked around an old base image can break on a newer one.**
  `v1.0.0` restored timezone data with `microdnf reinstall tzdata -y` in
  `append_final`, because older ee-minimal images shipped without zoneinfo.
  The current `:2.16` and `:2.17` images ship full zoneinfo, and the
  reinstall now exits with "The transaction was empty", failing the build at
  its very last step. `v1.0.1` drops it. Timezones still work, because the
  data is in the base image.

### `ee-unifi-rhel9`: Python swap in `prepend_base`

The unifi inventory plugin needs Python 3.13, but the base image ships 3.11.
The definition installs EPEL and Python 3.13, makes it the default
`python3`, and reinstalls Ansible on it, all in **`prepend_base`**. It has
to be base: the galaxy stage runs `ansible-galaxy`, builder compiles wheels
for the interpreter, and final installs those wheels. All three must see
the same Python, and only base reaches all of them.

### `ee-xorriso-rhel9`: RPMs from outside the default repos

xorriso isn't in the UBI repos, so its RPMs are downloaded and installed in
`prepend_base`. That works, and they reach final through base, but it's
only a **runtime** need. `append_final` would be the leaner choice, keeping
the RPMs out of the galaxy and builder stages.

### Corporate CA certificate (template example)

```yaml
build_items: [corp-ca.pem]
build_files:
  - src: corp-ca.pem
    dest: certs
build_steps:
  prepend_base:
    - COPY _build/certs/corp-ca.pem /etc/pki/ca-trust/source/anchors/corp-ca.pem
    - RUN update-ca-trust
```

`prepend_base`, because galaxy (hub over TLS), builder (pip over TLS) and
final (runtime connections) all need to trust it. It uses `_build/...`,
not `/build/...`, because `/build` doesn't exist in base.

### Private Automation Hub token (done for you)

```yaml
# Generated by infra.ee_utilities.ee_builder when ee_pull_collections_from_hub: true
additional_build_files:
  - src: ansible.cfg
    dest: configs
additional_build_steps:
  prepend_galaxy:
    - ADD _build/configs/ansible.cfg /etc/ansible/ansible.cfg
```

The token is used for the collection download in galaxy, and only
`/usr/share/ansible` is copied forward. So `/etc/ansible/ansible.cfg`, and
the token in it, isn't in the final image. This was confirmed on a built
image from this repo.

---

## See it for yourself

```bash
# 1. Render the ansible-builder definition for an EE and keep it on the VM
ansible-playbook build_ee.yml -e ee_file=ee_definitions/ee-demo-minimal_v1.0.0.yml \
  -e render_only=true -e keep_context=true

# 2. Generate the Containerfile from it (no build)
cd ~/ee-builds/ee-demo-minimal_v1.0.0
ansible-builder create -f ee-demo-minimal-execution-environment.yaml -c /tmp/ctx
less /tmp/ctx/Containerfile        # find your build_steps between the stages

# 3. Inspect a built image's layers
podman image history localhost/ee-demo-minimal:1.0.0
```

For full build logs, including each stage's commands, run the build with
`-e keep_context=true -e '{"ee_verbosity": 3}'`.

---

## Verified behaviour

Test build with ansible-builder 3.1.1 on
`ansible-automation-platform-27/ee-minimal-rhel9:2.16`. Each hook wrote a
marker file, two hooks set an `ENV`, and two system packages were declared.
Inspecting the final image showed:

| Test | Result in the final image |
| :-- | :-- |
| Files created in `prepend_base`, `append_base` | ✅ present |
| Files created in `prepend_final`, `append_final` | ✅ present |
| Files created in `prepend_galaxy`, `append_galaxy`, `prepend_builder`, `append_builder` (outside the copied paths) | ❌ absent |
| File created under `/usr/share/ansible` in `append_galaxy` | ✅ present |
| `ENV` set in `prepend_base` | ✅ set |
| `ENV` set in `prepend_galaxy` | ❌ not set |
| `jq [platform:rpm]` (untagged, runtime) | ✅ installed |
| `gcc [platform:rpm compile]` | ❌ absent |
| `gcc [platform:rpm]` (untagged) | ✅ installed, image +138 MB |
| `/output` build directory | ❌ removed |

References:
[ansible-builder definition format](https://ansible.readthedocs.io/projects/builder/en/stable/definition/) ·
[bindep profiles](https://docs.opendev.org/opendev/bindep/latest/readme.html) ·
[infra.ee_utilities ee_builder role](https://github.com/redhat-cop/ee_utilities/blob/devel/roles/ee_builder/README.md)
