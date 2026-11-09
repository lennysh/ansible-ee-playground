# Legacy example: a hand-maintained Containerfile + `podman build`

A realistic example of how many teams build EEs before adopting a
definition-driven workflow. A Containerfile was generated once by an old
ansible-builder, then edited by hand and built directly with `podman build`.

It's kept here to **compare** against the recommended approach. The same EE,
defined the recommended way, is
[`ee_definitions/examples/ee-winrm-kerberos-rhel8_v1.0.0.yml`](../../ee_definitions/examples/ee-winrm-kerberos-rhel8_v1.0.0.yml).
The results are in
**[docs/podman-build-vs-ee-definition.md](../../docs/podman-build-vs-ee-definition.md)**.

## What's here

| File | Purpose |
| :-- | :-- |
| `Containerfile` | The hand-maintained build: base, builder and final stages. Image references are generalized. |
| `bindep.txt` | System packages (RPMs) |
| `requirements.txt` | Python packages: `pywinrm[kerberos]`, `requests`, `jmespath` |
| `ca-bundle.crt` | **Example** corporate CA: a throwaway test certificate. Replace it with your own. |
| `pip.conf` | **Example** pip config pointing at public PyPI. Replace it with your internal mirror. |
| `krb5.conf` | **Example** Kerberos config for `EXAMPLE.COM`. Replace it with your AD realm and KDCs. |
| `compare.sh` | Builds this Containerfile **and** the recommended definition, then compares time, size and content |

## Building it the legacy way

The Containerfile expects a `_build/` directory that isn't stored here:
`_build/scripts/` holds ansible-builder 3.0.x helper scripts, alongside
`_build/requirements.txt` and `_build/bindep.txt`. That hidden dependency on
generated files is one of the problems with this approach. `compare.sh`
recreates it automatically.

It also only builds on a **subscribed RHEL host**: it deletes the UBI repo
file, and `git-subtree` and `python3-devel` are only in RHEL repos.
`compare.sh` applies three small substitutions so it builds on any podman
host. They're documented at the top of the script.

## Running the comparison

On the EE Builder VM, from the repo root:

```bash
podman login registry.redhat.io
ROUNDS=2 examples/legacy-podman-build/compare.sh | tee compare-results.md
```
