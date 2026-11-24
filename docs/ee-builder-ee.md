# An EE that builds EEs (`ee-builder-rhel9`)

Normally `build_ee.yml` runs from AAP and SSHes to a dedicated **EE Builder
VM** that has `ansible-builder` and `podman` installed (see
[aap/README.md](../aap/README.md)). This guide covers the alternative: an
**execution environment that contains the build tooling itself**. The build
job runs in that EE, and podman builds the new EE **inside the job
container** ("podman in podman"), then pushes it to your registry or
Private Automation Hub. No builder VM is needed.

The same EE works:

* **locally**, with plain rootless `podman run` (or `ansible-navigator`)
* **in AAP on VM-based execution nodes**, with **no changes to AAP
  settings** and SELinux left enforcing
* **in AAP container groups on OpenShift**, in a pod that has a user
  namespace (see [OpenShift](#openshift-container-groups))

`build_ee.yml` doesn't change. Only the inventory is different: `localhost`
(the job container) instead of the builder VM.

| | EE Builder VM (default) | Builder EE (this guide) |
| :-- | :-- | :-- |
| Where the build runs | Builder VM, over SSH | Inside the job container |
| Extra infrastructure | A RHEL VM, SSH credential | None |
| Layer cache between builds | Yes (VM's podman storage) | No: every job starts empty and pulls base images again |
| Disk used during a build | VM home directory | The execution node's (or pod's) container storage |

- [The definitions](#the-definitions)
- [Requirements for the job container](#requirements-for-the-job-container)
- [Using it locally](#using-it-locally)
- [Using it from AAP (VM-based execution nodes)](#using-it-from-aap-vm-based-execution-nodes)
- [OpenShift (container groups)](#openshift-container-groups)
- [Verified behaviour](#verified-behaviour)
- [Troubleshooting](#troubleshooting)

---

## The definitions

| File | Base image | Use it for |
| :-- | :-- | :-- |
| [`ansible-automation-platform-27/ee-builder-rhel9_v1.0.0.yml`](../ee_definitions/ansible-automation-platform-27/ee-builder-rhel9_v1.0.0.yml) | `ansible-automation-platform-27/ansible-dev-tools-rhel9:26.11.0` | **Recommended.** Red Hat's dev-tools image, plus the collections `build_ee.yml` needs |
| [`examples/ee-builder-ubi-rhel9_v1.0.0.yml`](../ee_definitions/examples/ee-builder-ubi-rhel9_v1.0.0.yml) | `ansible-automation-platform/ee-minimal-rhel9:2.17` | Reference: the same capability built from scratch, showing every piece of podman-in-podman setup |

Build either one like any other EE:

```bash
ansible-playbook build_ee.yml --ask-vault-pass \
  -e ee_file=ee_definitions/ansible-automation-platform-27/ee-builder-rhel9_v1.0.0.yml
```

### Why the ansible-dev-tools base

`ansible-dev-tools-rhel9` ships ansible-core, ansible-runner,
ansible-builder and podman, and Red Hat sets it up for rootless nested
podman. This repo adds two things: the collections `build_ee.yml` needs and
`BUILDAH_ISOLATION=chroot`.

> The AAP 2.7 image (`26.11.0`) is **not itself an EE**: it has no EE
> entrypoint and no `ansible-execution-environment` label. (The AAP 2.6
> `26.1.0` image did.) That doesn't matter here: ansible-builder's final
> stage adds the entrypoint, `dumb-init`, `/runner` and the label to every
> image it builds, so `ee-builder-rhel9` is a proper EE. Don't use the
> dev-tools image directly as an AAP EE.

| Needed for nested builds | ansible-dev-tools base | From scratch (`examples/`) |
| :-- | :-- | :-- |
| `ansible-builder` | included (3.1.x) | pip `ansible-builder` |
| `podman`, `fuse-overlayfs`, `crun`, `passt` | included | bindep `system:` packages |
| `netavark` | included | **Not installable from UBI as-is:** it requires `aardvark-dns`, which is only in entitled RHEL repos. Installed with `rpm --nodeps` in `append_base` (aardvark-dns only provides DNS on custom podman networks, which nested builds don't use) |
| `/etc/subuid`, `/etc/subgid` for uid 1000 | included (user `podman`) | written in `append_final`, keyed `1000` (the user name the runtime creates for uid 1000) |
| `containers.conf`: share namespaces, no cgroups | included | written in `append_final` |
| `storage.conf` | included | written in `append_final` (fuse-overlayfs) |
| `BUILDAH_ISOLATION=chroot` | **added by this repo** | added |
| Collections `infra.ee_utilities`, `containers.podman` | **added by this repo** | added |
| ansible-core | 2.16 | 2.17 |
| Image size | ~590 MB | ~590 MB |

## Requirements for the job container

The inner podman runs **rootless** as the EE user (uid 1000), and image
builds use **chroot isolation**, so `RUN` steps need no mounts or
privileges. The job container needs:

* **Run as uid 1000** (the image default) so the `/etc/subuid` range
  applies. Inner podman then gets 65,536 ids:
  `0→1000, 1→1 ×999, 1000→1001 ×64535`.
* **A range of sub-uids for the outer container.** Rootless podman on AAP
  nodes and workstations has this by default. On OpenShift it means a user
  namespace.

It **doesn't** need `--privileged`, `--device /dev/fuse`,
`--security-opt label=disable` or any other extra run options.

> `BUILDAH_ISOLATION=chroot` makes the difference. Without it (the stock
> dev-tools image), buildah's default isolation uses bind mounts that SELinux
> blocks (`pkg/bind: error detaching ... permission denied`, plus SELinux
> `mounton` alerts on the host). The only workaround then is
> `--security-opt label=disable`.

To also **run** a freshly built image inside the job (e.g. `podman run` as a
smoke test), the job container additionally needs
`--security-opt unmask=ALL`, because inner containers mount their own
`/proc`. Building and pushing don't need it.

## Using it locally

From the repo root. Mount your registry login read-only so the build can pull
base images and push. The file has to be readable by uid 1000 inside the
container, so make a temporary copy:

```bash
install -m 0644 ${XDG_RUNTIME_DIR}/containers/auth.json /tmp/ee-auth.json

podman run --rm -it \
  -v "$PWD":/runner/project:z \
  -v /tmp/ee-auth.json:/tmp/auth.json:ro,z -e REGISTRY_AUTH_FILE=/tmp/auth.json \
  quay.io/lshirley/ansible-automation-platform-27/ee-builder-rhel9:1.0.0 \
  bash -c 'cd /runner/project && ansible-playbook -i inventory.yml build_ee.yml \
           --ask-vault-pass -e ee_file=ee_definitions/ee-demo-minimal_v1.0.0.yml'

rm /tmp/ee-auth.json
```

* `inventory.yml` already targets `localhost` with `ansible_connection: local`.
* Credentials from `config/vault.yml` (`vault_registry_credentials`) work as
  usual. The role logs in with them. The mounted auth file is only needed
  for registries that aren't in the vault.
* For a registry on the host itself (e.g. `localhost:5000`), add
  `--network host`.

## Using it from AAP (VM-based execution nodes)

1. **Build and push** `ee-builder-rhel9`, using the existing
   *Build Execution Environment* job template or the CLI.
2. **Create the EE in AAP**: *Automation Execution > Infrastructure >
   Execution Environments*, image
   `quay.io/lshirley/ansible-automation-platform-27/ee-builder-rhel9:1.0.0`, plus
   a registry credential if the repository is private.
3. **Inventory**: create e.g. *EE Builder (in EE)* with group `ee_builders`
   containing `localhost`, with host vars:

   ```yaml
   ansible_connection: local
   ansible_python_interpreter: "{{ ansible_playbook_python }}"
   ```

   This is the same as [inventory.yml](../inventory.yml): the "builder" is
   the job container itself.
4. **Job template**: copy *Build Execution Environment* and change:
   * **Inventory**: the one from step 3
   * **Execution environment**: `ee-builder-rhel9`
   * **Credentials**: *EE Builder Secrets* only (no machine/SSH credential)

   The survey and extra vars stay the same.
5. **Disk space**: inner container storage lives in anonymous volumes on the
   execution node and is removed when the job ends (AAP runs jobs with
   `--rm`). Each build needs several GB free on the node while it runs.

You don't need to change *Settings > Job > Container Run Options*.
ansible-runner starts the job container with the image's user (uid 1000),
`--group-add=root` and `--ipc=host`, which is exactly the setup tested here.

## OpenShift (container groups)

**What doesn't work: default random-uid pods** (`restricted-v2` SCC). The pod
runs as an arbitrary uid with no sub-uid range, so inner podman falls back to
a single-uid mapping. Base images can be pulled, but the first `RUN` step of
any build fails with `error setting supplemental groups list: operation not
permitted`, because buildah can't call `setgroups` in a single-uid user
namespace. This is a limitation of rootless podman, not of this EE.

**What should work: a pod with its own user namespace** running as uid
1000. Inside, that's identical to the tested AAP-node setup. On OpenShift
4.14+ this means a container group with a custom pod spec along these lines,
plus the `nested-container` SCC (or equivalent) granted to the pod's service
account:

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: aap
spec:
  hostUsers: false              # user namespace for the pod
  serviceAccountName: default   # must be allowed to use the nested-container SCC
  automountServiceAccountToken: false
  containers:
    - image: quay.io/lshirley/ansible-automation-platform-27/ee-builder-rhel9:1.0.0
      name: worker
      args: [ansible-runner, worker, --private-data-dir=/runner]
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        procMount: Unmasked
      resources:
        requests:
          cpu: 250m
          memory: 1Gi
          ephemeral-storage: 10Gi
```

> **Not tested here.** The details (SCC name, annotations, whether
> `/dev/fuse` must be exposed) vary by OpenShift version. Check Red Hat's
> OpenShift docs on running nested containers in user namespaces for your
> version.

## Verified behaviour

Tested on Fedora with rootless podman 5.8 and **SELinux enforcing**:

| Scenario | Job container options | Result |
| :-- | :-- | :-- |
| AAP-node style: `build_ee.yml` inside `ee-builder-rhel9`, inventory `localhost` | `--group-add=root --ipc=host` (what ansible-runner adds) | Built `<name>:<version>` + `:latest`, **pushed** to a plain-HTTP registry (`ee_validate_certs: false`) and to a real TLS registry. 0 failed tasks |
| Same, from-scratch UBI variant (`ansible-builder build`) | none | Builds |
| OpenShift-style random uid | `--user 123456:0` | **Fails** at the first `RUN` step (`setgroups`), as described above |
| Stock dev-tools image, no `BUILDAH_ISOLATION=chroot` | none | **Fails** (`pkg/bind ... permission denied`, SELinux `mounton`). Works with `--security-opt label=disable` |
| `podman run` of a built image inside the job | none / `--security-opt unmask=ALL` | Fails (`mount proc`) / works |

## Troubleshooting

| Symptom | Cause / fix |
| :-- | :-- |
| `pkg/bind: error detaching ... permission denied`, SELinux `mounton` alerts | Not using chroot isolation. Make sure `BUILDAH_ISOLATION=chroot` is set (it is in both definitions). |
| `error setting supplemental groups list: operation not permitted` | Single-uid mode: the job's uid has no `/etc/subuid` range (random uid, or not uid 1000). Run as uid 1000 with a user namespace. |
| `potentially insufficient UIDs or GIDs available in user namespace` (when starting the job container) | The outer runtime can't map the requested `--user` uid. Use the image default (1000). |
| `newuidmap: open of uid_map failed: Permission denied` | `newuidmap`/`newgidmap` were made **setuid**. Leave them with their file capabilities (`cap_setuid`/`cap_setgid`). Don't `chmod u+s` them. |
| `nothing provides aardvark-dns` while installing podman | UBI repos lack `aardvark-dns`. Use the dev-tools base, or see the `append_base` workaround in the from-scratch example. |
| `crun: mount proc to proc: Operation not permitted` | Running (not building) a container inside the job. Add `--security-opt unmask=ALL` to the job container. |
| `reading JSON file ... auth.json: permission denied` | The mounted auth file isn't readable by uid 1000 inside the container (`chmod 0644` a temporary copy). |
