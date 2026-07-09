# harvester-molecule-kubevirt

A minimal Molecule scenario that creates ephemeral VMs on **SUSE Virtualization / Harvester** through the KubeVirt API, using only `kubernetes.core` and `community.crypto` - no Molecule driver plugin.

Adapted from the upstream [Molecule KubeVirt example](https://docs.ansible.com/projects/molecule/examples/kubevirt/), with fixes for Harvester specifics (see [Deviations](#deviations-from-upstream)).

Lifecycle: `create` -> `converge` -> `idempotence` -> `verify` -> `destroy`.
Connectivity: SSH over a `NodePort` Service, keyed by a throwaway SSH key generated at `create` time.

---

## Layout

```
.
├── README.md
├── GEMINI.md                    # agent operating instructions
├── Taskfile.yml                 # task deps / check / test / destroy
├── ansible.cfg
├── requirements.txt             # python deps
├── requirements.yml             # galaxy collections
├── .env.example
├── manifests/
│   └── rbac.yaml                # optional least-privilege ServiceAccount
└── molecule/
    └── harvester/
        ├── molecule.yml         # platforms + test_sequence
        ├── requirements.yml
        ├── create.yml
        ├── converge.yml
        ├── verify.yml
        ├── destroy.yml
        └── tasks/
            ├── create_vm.yml
            └── create_vm_dictionary.yml
```

---

## Prerequisites

| Item | Requirement |
|---|---|
| Harvester | Any release exposing `kubevirt.io/v1` `VirtualMachine` |
| Access | Harvester kubeconfig, or a ServiceAccount token + API URL |
| Network | NodePort range `30000-32767` reachable from your workstation to Harvester node IPs |
| Registry | Egress to `quay.io`, or a mirrored containerdisk in a local registry |
| Python | 3.10+ |

The VMs are **ephemeral** - `containerDisk` + `emptyDisk`, no PVC, no Longhorn volume. Nothing survives a reboot. This is deliberate: it removes storage class and VirtualMachineImage from the test surface.

---

## Install

```bash
python3 -m venv .venv && source .venv/bin/activate
cp .env.example .env        # then edit
task deps
```

Get a kubeconfig from the Harvester UI: **Support -> Download KubeConfig**. Save it as `harvester.kubeconfig` in the repo root (it is gitignored).

Authentication, pick one:

**A. kubeconfig** (simplest)

```bash
export KUBECONFIG="$PWD/harvester.kubeconfig"
```

**B. ServiceAccount token** (matches the upstream example)

```bash
task rbac
export K8S_AUTH_HOST="https://<harvester-vip>:6443"
export K8S_AUTH_API_KEY="$(task token --silent)"
export K8S_AUTH_VERIFY_SSL="false"
```

`K8S_AUTH_VERIFY_SSL=false` disables TLS verification. Fine for a lab against a self-signed Harvester cert, not fine anywhere else - prefer pinning the CA via kubeconfig.

Sanity check before running anything:

```bash
task check
```

---

## Usage

```bash
task test        # full sequence, tears down on exit
```

Iterating:

```bash
task create      # VMs up, inventory written
task converge    # apply your role
task login       # ssh into mol-ubuntu2404
task verify      # assertions
task destroy     # clean up
```

If `molecule test` dies mid-run and leaves objects behind:

```bash
task ns-clean
```

---

## Configuration

Everything is driven from `molecule/harvester/molecule.yml`.

| Key | Meaning |
|---|---|
| `platforms[].name` | VM name, Service name, and inventory hostname |
| `platforms[].image` | containerdisk OCI ref |
| `platforms[].namespace` | Harvester namespace; created if absent |
| `platforms[].ansible_user` | cloud-init user that receives the temporary SSH key |
| `platforms[].memory` | `spec.domain.resources.requests.memory` |
| `platforms[].cpu_cores` | `spec.domain.cpu.cores` |
| `platforms[].capacity` | size of the ephemeral `emptyDisk` |
| `harvester_nodeport_host_override` | Set when node InternalIPs are not routable from your workstation. Skips the `nodes` read entirely. |

### Swapping the guest image

Available images under `quay.io/containerdisks/`: `fedora`, `ubuntu`, `centos-stream`, `debian`, `opensuse-leap`, `opensuse-tumbleweed`, and others. Change `image` **and** `ansible_user` together - the cloud user differs per distro (`ubuntu`, `fedora`, `debian`, ...).

`[VERIFY]` Exact tags. Check with `skopeo list-tags docker://quay.io/containerdisks/ubuntu` before pinning.

### Air-gapped

Mirror the containerdisk into your Harbor and point `image` at it. Nothing else in the scenario reaches the internet at runtime; `task deps` does, so pre-seed the venv and `collections/` on the connected side.

---

## Deviations from upstream

The upstream example does not run cleanly against Harvester. Changes:

| # | Upstream | Here | Why |
|---|---|---|---|
| 1 | `ansible_host: <node name>` | `ansible_host: <node InternalIP>` | Harvester node names rarely resolve from a workstation. Costs a cluster-scoped `nodes` get - override with `harvester_nodeport_host_override` to avoid it. |
| 2 | `nodeport_host` is a scalar set inside a loop | `molecule_nodeport_hosts` dict keyed by VM name | Upstream overwrites the fact each iteration, so every host ends up pointing at the last VM's node. |
| 3 | `when: vm.ssh_service.type == 'NodePort'` on a `block` | `when` on the looped task, using the loop var | `vm` is not in scope at block level; it only works by accident via fact leakage from the previous `include_tasks` loop. |
| 4 | No `interfaces` / `networks` | explicit `masquerade` on the pod network | Harvester's UI injects a network config; API-created VMs get none. |
| 5 | `runcmd: yum install qemu-guest-agent` | removed | Distro-specific, needs egress, and the guest agent is not required for SSH-based Molecule. |
| 6 | No wait | `wait_condition: {type: Ready}` on the VM, retry loop on the virt-launcher pod | Otherwise `create` races the scheduler. |
| 7 | `callback_whitelist` | `callbacks_enabled` | The former is removed in ansible-core 2.16+. |
| 8 | `ansible_ssh_port` | `ansible_port` | Both work; `ansible_port` is the canonical name. |
| 9 | Namespace assumed to exist | created idempotently in `create.yml` | |

Also note: `spec.running: true` is deprecated in favour of `spec.runStrategy`. Harvester's UI writes `runStrategy: RerunOnFailure`. This scenario keeps `running: true` because it is what upstream KubeVirt still accepts everywhere. `[VERIFY]` on your Harvester version before switching - never set both fields.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `wait_for_connection` times out | NodePort unreachable. `kubectl -n molecule get svc` for the port, then `nc -vz <node-ip> <nodePort>`. |
| VM stuck `Scheduling` | Image pull failing. `kubectl -n molecule describe pod -l vm.kubevirt.io/name=<vm>` |
| `Forbidden: virtualmachines.kubevirt.io` | RBAC. Re-run `task rbac`, or use an admin kubeconfig. |
| `nodes is forbidden` | ClusterRole not bound, or set `harvester_nodeport_host_override`. |
| SSH auth fails | cloud-init did not apply. `virtctl console <vm>` and check `/var/log/cloud-init-output.log`. |
| Idempotence fails | Your `converge.yml` has a non-idempotent task, not a scenario bug. |
