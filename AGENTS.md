# AGENTS.md

Operating instructions for an AI agent working in this repository.

## What this repo is

A single Molecule scenario (`molecule/harvester`) that provisions ephemeral KubeVirt VMs on a SUSE Virtualization (Harvester) cluster and tests an Ansible role against them over SSH via a NodePort Service.

There is no Molecule driver plugin. All lifecycle logic lives in `create.yml` / `destroy.yml` and uses `kubernetes.core`.

## Ground rules

1. **This repo talks to a live cluster.** Every `molecule create`, `converge`, `test`, or `kubectl apply` mutates real infrastructure. Never run them without an explicit instruction in the current turn.
2. **Never run `destroy`, `test`, or `task ns-clean` against a namespace you did not create.** Confirm `kubectl config current-context` and the `namespace` field in `molecule.yml` first.
3. **Never commit or print** `harvester.kubeconfig`, `.env`, `K8S_AUTH_API_KEY`, or any token. They are gitignored; keep it that way.
4. **Plan before writing.** When asked to change behaviour, state the plan and the files involved. Do not generate or edit files during a planning turn.
5. **Do not invent Harvester or KubeVirt API fields.** If a field is not in the KubeVirt v1 `VirtualMachine` schema you can point to, mark it `[VERIFY]` and say so.

## Uncertainty markers

Use these inline, in code comments and in prose. They are load-bearing.

| Marker | Meaning |
|---|---|
| `[VERIFY]` | Architectural inference, not a documented fact. Needs a primary source (KubeVirt API ref, Harvester docs, a GitHub issue) before it is trusted. |
| `[MANUAL]` | Requires a human to run a command or make a decision. |
| `[GATE]` | A hard stop. Nothing downstream proceeds until this is validated. |

## Repo map

| Path | Role |
|---|---|
| `molecule/harvester/molecule.yml` | Only file most users should edit. Platforms, test sequence, group_vars. |
| `molecule/harvester/create.yml` | Keypair, namespace, VMs, NodePort Services, inventory. |
| `molecule/harvester/tasks/create_vm.yml` | The VirtualMachine manifest and node-IP resolution. |
| `molecule/harvester/tasks/create_vm_dictionary.yml` | Builds the `molecule_systems` inventory dict. |
| `molecule/harvester/converge.yml` | Where the role under test goes. |
| `molecule/harvester/verify.yml` | Assertions. |
| `molecule/harvester/destroy.yml` | Deletes VM + Service, waits for the virt-launcher pod to vanish. |
| `manifests/rbac.yaml` | Optional least-privilege SA. |
| `Taskfile.yml` | The only supported entry point. Do not suggest raw `molecule` commands when a task exists. |

## Invariants

Breaking any of these breaks the scenario. Check them after every edit.

- `platforms[].name` is used as the VM name, the Service name, the `kubevirt.io/domain` label, the Service selector, and the inventory hostname. All five must stay in sync.
- `converge.yml` asserts `hostname == inventory_hostname`. cloud-init `hostname` must therefore equal `vm.name`.
- The Service selector matches `kubevirt.io/domain`, which is set on **both** `metadata.labels` and `spec.template.metadata.labels`. The latter is the one that lands on the virt-launcher pod. Do not remove it.
- `molecule_nodeport_hosts` is a dict keyed by VM name. Do not collapse it back to a scalar - that is upstream bug #2 in the README.
- `spec.running` and `spec.runStrategy` are mutually exclusive. Never set both.
- `node_port_services.results` contains skipped entries when a platform is not `NodePort`. `create_vm_dictionary.yml` filters with `rejectattr('skipped', 'defined')`. Preserve that filter if you add non-NodePort platforms.

## Commands

Read-only, safe to run when investigating:

```bash
task check
kubectl -n molecule get vm,vmi,svc,pod -o wide
kubectl -n molecule describe vm <name>
kubectl -n molecule logs -l vm.kubevirt.io/name=<name> -c compute
ansible-lint
molecule syntax -s harvester
```

Mutating, requires explicit instruction:

```bash
task create | converge | verify | destroy | test | rbac | ns-clean
```

## Debug loop

When a run fails:

1. Ask for the **full terminal output**, not a summary.
2. Classify: RBAC (`Forbidden`), scheduling (`Pending` / `ImagePullBackOff`), network (`wait_for_connection` timeout), cloud-init (SSH auth failure), or playbook logic.
3. Reproduce the smallest failing step. `molecule create -s harvester` alone, not `molecule test`.
4. Deliver the **corrected file in full**, not a diff narrative.
5. Leave the cluster clean: if you created objects while debugging, `task destroy`.

## Style

- Terse. Copy-paste ready. No padding, no restating the request.
- Code blocks for all YAML, commands, and manifests.
- Tables over bullet lists for anything enumerable.
- Dashes, not semicolons. No em-dashes.
- Comment *why*, never *what*. `# Harvester injects no network for API-created VMs` is useful; `# set the image` is not.
- Ansible: FQCN for every module. `ansible.builtin.set_fact`, not `set_fact`.
- YAML: two-space indent, `---` document start, quoted booleans only where the k8s API needs a string (`status: "True"`).

## Extending

| Want | Do |
|---|---|
| Test a real role | Replace the tasks in `converge.yml` with `roles: [<name>]`, put the role in `roles/`, set `ANSIBLE_ROLES_PATH`. |
| Persistent disk | Swap `containerDisk` for a `dataVolumeTemplate` backed by a Harvester `VirtualMachineImage`. This pulls storage class + Longhorn into scope. `[GATE]` confirm the target storage class first. |
| VLAN network | Add a Multus `NetworkAttachmentDefinition` reference under `spec.template.spec.networks` and a `bridge` interface. NodePort SSH then becomes optional. `[VERIFY]` Harvester requires the NAD to exist in the VM's namespace or in `default`. |
| Second scenario | Copy `molecule/harvester/` to `molecule/<name>/`; `SCENARIO` in `Taskfile.yml` is the switch. |
