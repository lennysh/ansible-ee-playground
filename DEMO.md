# Demo script: building EEs from the EE Builder VM

A ~30 minute walkthrough in two parts:

* **Part 1 (CLI, ~20 min):** on the EE Builder VM, to show how it works
* **Part 2 (AAP, ~10 min):** the same build from an AAP job template,
  which is how it will be run day to day

CLI commands are run **on the EE Builder VM** from the repo root. Lines
starting with 💬 are suggested talking points.

**Before the demo:**

- [ ] Complete the [one-time setup](README.md#one-time-setup) (settings, vault, collections).
- [ ] Pre-pull the base image so the live build is quick:
      `podman pull registry.redhat.io/ansible-automation-platform/ee-minimal-rhel9:2.16`
- [ ] Optional: build `ee-demo-minimal_v1.0.0` once beforehand so collection
      downloads are cached and you have a fallback.
- [ ] Have the Private Automation Hub UI open in a browser tab (Execution Environments page).
- [ ] Create `.vault_pass` and enable `vault_password_file` in your local `ansible.cfg`,
      so you don't type the vault password during the demo.
- [ ] For Part 2: AAP is configured ([aap/README.md](aap/README.md)), the
      repo with your demo commits is pushed, and the job template has run
      once successfully.

---

# Part 1: CLI on the EE Builder VM

---

## 1. The big idea (2 min)

```bash
tree -L 2 -I collections
```

💬 *"Each EE you own is one YAML file in `ee_definitions/`. The file name is
the image name plus its version. There's one playbook, and the build logic
comes from the supported Red Hat CoP collection `infra.ee_utilities`, so
there's no build code here for you to maintain."*

```bash
ls ee_definitions/
```

## 2. Anatomy of an EE file (3 min)

```bash
cat ee_definitions/ee-demo-minimal_v1.0.0.yml
```

💬 Point out:

- `destination`: **this EE decides where it's published** (registry +
  namespace). Different EEs can go to different registries.
- `base_image`: a supported Red Hat image, pinned to a tag.
- `dependencies.galaxy`: collections, pulled from **your** Private Automation Hub.
- `dependencies.python` / `dependencies.system`: pip and RPM packages.
- There's **no name or version inside the file**. It comes from the filename.

Then a more realistic one:

```bash
tree ee_definitions/
cat ee_definitions/ansible-automation-platform-26/ee-casc-rhel9_v1.0.0.yml
```

💬 *"Folders keep things organised. Here they mirror the destination
namespaces: multi-version EEs go to `.../ansible-automation-platform`,
2.6-only EEs to `.../ansible-automation-platform-26`. Version ranges on the
certified collections keep this EE matched to AAP 2.6. `build_steps` lets
you run any Containerfile command, e.g. adding a CA cert."*

## 3. Global settings and secrets (2 min)

```bash
cat config/settings.yml
ansible-vault view config/vault.yml     # encrypted at rest; show the STRUCTURE, not real passwords
```

💬 *"Settings shared by every EE live in one place: which hub to pull
collections from, the default base image, and builder options. Where an EE
is pushed is **not** here; that's in each EE file.*

*Registry credentials are one map keyed by registry host. The playbook
uses the destination's host to choose push credentials and the base
image's host to choose pull credentials. Add a new registry once and every
EE that uses it just works. The vault is never committed, and the hub token
is used only during the build. It is **not** baked into the final image."*

## 4. Preview without building (2 min)

```bash
ansible-playbook build_ee.yml -e ee_file=ee_definitions/ee-demo-minimal_v1.0.0.yml -e render_only=true
```

Scroll to **Show rendered execution-environment.yaml** in the output.

💬 *"This is the standard ansible-builder v3 file the collection generates.
You can review it in a merge request before you spend time on a build."*

## 5. Build and push (5 min)

```bash
ansible-playbook build_ee.yml -e ee_file=ee_definitions/ee-demo-minimal_v1.0.0.yml
```

💬 While it runs, walk through the output:

1. **Show build plan**: name, version and tags come from the filename, and
   the destination from the file. *pull auth* / *push auth* show which
   registry's stored credentials are used (never the values).
2. **Check builder tooling**: confirms podman and ansible-builder are present.
3. **Pull base image**: always fresh, so it picks up the latest CVE fixes.
4. **Run the Ansible Builder program**: the actual build.
5. **Push image to registry**: pushes `:1.0.0` **and** `:latest`.

Show the result:

```bash
podman images | grep ee-demo-minimal
```

➡️ Switch to the **Private Automation Hub UI → Execution Environments** and show
`ee-demo-minimal` with both tags.

## 6. Test the EE (2 min)

```bash
podman run --rm localhost/ee-demo-minimal:1.0.0 ansible-galaxy collection list
podman run --rm localhost/ee-demo-minimal:1.0.0 jq --version
podman run --rm localhost/ee-demo-minimal:1.0.0 \
  ansible localhost -m ansible.builtin.debug \
  -a 'msg={{ "192.168.1.10/24" | ansible.utils.ipaddr("network") }}'
```

💬 *"The collection, the Python library and the RPM are all in the image and
working."*

## 7. Release a new version (4 min)

💬 *"Now a team asks for `ansible.posix`. Here's the change process."*

```bash
git mv ee_definitions/ee-demo-minimal_v1.0.0.yml ee_definitions/ee-demo-minimal_v1.1.0.yml
```

Edit the file and add under `collections:`:

```yaml
      - name: ansible.posix
```

```bash
git diff --staged -M; git diff
git commit -am "ee-demo-minimal 1.1.0: add ansible.posix"
ansible-playbook build_ee.yml -e ee_file=ee_definitions/ee-demo-minimal_v1.1.0.yml
```

➡️ In the hub UI: `1.0.0`, `1.1.0`, and `latest` now points to `1.1.0`.

💬 *"Jobs pinned to `:1.0.0` keep working unchanged. You move them to `:1.1.0`
when you're ready. Git history plus the filename tell you exactly what's in
every tag."*

## 8. Guardrails (1 min)

```bash
cp ee_definitions/ee-demo-minimal_v1.1.0.yml ee_definitions/MyEE.yml
ansible-playbook build_ee.yml -e ee_file=ee_definitions/MyEE.yml -e render_only=true
rm ee_definitions/MyEE.yml
```

💬 *"Bad names fail straight away with a clear message. The same happens if
someone tries to set `name` or `tags` inside a file."*

## 9. Creating a brand new EE (1 min)

```bash
less templates/ee-name_v0.1.0.yml
```

💬 *"To add a new EE, copy this template into `ee_definitions/`, rename it,
and delete what you don't need. Every option is documented inline, including
common needs like corporate CA certificates."*

---

# Part 2: the same build from AAP

💬 *"Day to day, nobody needs to log in to the builder VM. The same playbook
and the same EE files run from an AAP job template."*

## 10. What's configured in AAP (2 min)

Show in the AAP UI:

- **Project** *Ansible EE Playground*: it's this git repo. Point out
  *Update revision on launch*, so every build uses the latest merged files.
- **Inventory** *EE Builders*: the builder VM in group `ee_builders`.
- **Credentials**: *EE Builder VM SSH*, plus *EE Builder Secrets*, a custom
  credential type. Open it to show the secret fields are hidden.

💬 *"The secrets live in AAP and are injected at run time. Users who can
launch a build never see them."*

```bash
less aap/vars/ee_builder_aap.yml
```

💬 *"All of this AAP setup is code too, applied with
`infra.aap_configuration`. Re-running it also refreshes the survey's list
of EEs."*

## 11. Launch a build (5 min)

**Templates → Build Execution Environment → Launch**

- *EE definition file*: `ee_definitions/ee-demo-minimal_v1.1.0.yml`
  (the version created in step 7; re-run `aap/configure_aap.yml` first so
  it's in the list)
- *Preview only?*: first `true`, to show the rendered file in the job
  output. Then launch again with `false`.
- *Push to registry?*: `true`

💬 While it runs, point out the same tasks as the CLI run: build plan,
ansible-builder, push. It's the same playbook.

## 12. Use the new EE (2 min)

**Infrastructure → Execution Environments → Create**:
image `pah.example.com/custom-ees/ee-demo-minimal:1.1.0`, with the hub
registry credential. Attach it to any job template.

💬 *"From a merged file to a usable EE in AAP, all through the UI and fully
audited: who built what, when, from which commit."*

## 13. Wrap-up: where this goes next

- **Self-service:** give teams *Execute* on the job template. The platform
  team reviews EE changes in merge requests.
- **Automate it:** trigger the job template from CI on merge to `main`.
- **Patch cadence:** bump the PATCH version to rebuild on fresh base images
  and pick up CVE fixes.

---

## Cleanup after the demo

```bash
git reset --hard HEAD~1              # undo the 1.1.0 demo commit, if you don't want to keep it
podman rmi localhost/ee-demo-minimal:1.0.0 localhost/ee-demo-minimal:1.1.0 localhost/ee-demo-minimal:latest
rm -rf ~/ee-builds/
```
