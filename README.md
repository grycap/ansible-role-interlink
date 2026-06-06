# ansible-role-interlink

Ansible role to install and configure InterLink components for Slurm and Kubernetes.

The role supports two execution modes:

- `interlink_type_of_node: slurm`: installs InterLink API + Slurm plugin, generates mTLS certificates, and exports client artifacts.
- `interlink_type_of_node: kubernetes`: deploys the InterLink Virtual Kubelet via Helm using artifacts generated in Slurm mode.

## What This Role Does

### Slurm mode

- Installs required packages (`ca-certificates`, `curl`, `jq`, `openssl`).
- Creates InterLink directories under `interlink_base_dir`.
- Downloads InterLink binaries and installs them under `interlink_base_dir/bin`.
- Generates CA/server/client mTLS certificates with `community.crypto`.
- Templates InterLink and Slurm plugin configuration files.
- Templates and manages systemd units:
  - `interlink-api.service`
  - `interlink-slurm-plugin.service`
- Validates local mTLS endpoint (`/pinglink`).
- Exports Kubernetes client cert artifacts to:
  - `{{ interlink_artifact_dir }}/ca.crt`
  - `{{ interlink_artifact_dir }}/tls.crt`
  - `{{ interlink_artifact_dir }}/tls.key`

### Kubernetes mode

- Verifies mTLS artifacts exist on the Ansible controller.
- Copies artifacts to target host (`interlink_remote_cert_dir`).
- Ensures namespace exists.
- Creates or updates TLS secret for Virtual Kubelet.
- Templates Helm values and installs/upgrades InterLink chart from GHCR OCI.
- Waits for rollout.
- Finds and approves pending kubelet-serving CSRs for the InterLink service account.
- Waits for Virtual Kubelet node Ready.
- Optionally runs smoke test pod (`interlink_run_smoke_test: true`).

## Requirements

- Ansible >= `2.12`
- Collection:
  - `community.crypto`
- External tools expected on target hosts:
  - Slurm mode: `curl`, `openssl`, `jq` (installed by role), systemd available.
  - Kubernetes mode: `kubectl`, `helm`, `jq` available in PATH.
- Access to Kubernetes admin kubeconfig (`kubeconfig_path`, default `/etc/kubernetes/admin.conf`).

## Role Variables

Defaults are defined in `defaults/main.yml`.

### Core mode and identity

- `interlink_type_of_node` (default: `kubernetes`): `kubernetes` or `slurm`.
- `type_of_node`: backward-compatible alias.
- `interlink_namespace` (default: `interlink`)
- `interlink_node_name` (default: `interlink-slurm-vk`)
- `interlink_run_smoke_test` (default: `false`)

### Paths and runtime

- `interlink_base_dir` (default: `/opt/interlink`)
- `interlink_data_dir` (default: `{{ interlink_base_dir }}/data`)
- `interlink_work_dir` (default: `{{ interlink_base_dir }}/work`)
- `interlink_home_dir` (default: `{{ interlink_base_dir }}/home`)
- `interlink_artifact_dir` (default: `{{ playbook_dir }}/artifacts/{{ interlink_node_name }}`)
- `interlink_remote_cert_dir` (default: `/tmp/{{ interlink_node_name }}-certs`)
- `kubeconfig_path` (default: `/etc/kubernetes/admin.conf`)

### Versions and image

- `interlink_version` (default: `0.6.1-patch1`)
- `interlink_slurm_plugin_version` (default: `0.6.1`)
- `interlink_chart_version` (default: `0.6.1`)
- `virtual_kubelet_image` (default: `ghcr.io/interlink-hq/interlink/virtual-kubelet-inttw:0.6.1-patch1`)

### Capacity and resource settings

- `virtual_node_cpu` (default: `1`)
- `virtual_node_memory_gib` (default: `1`)
- `virtual_node_pods` (default: `10`)
- `virtual_node_accelerators` (default: `[]`): explicit Helm `virtualNode.resources.accelerators` list. If unset and a Slurm capability artifact reports GPUs, the Kubernetes phase advertises `nvidia.com/gpu` automatically.
- `virtual_kubelet_cpu_request` (default: `100m`)
- `virtual_kubelet_cpu_limit` (default: `500m`)
- `virtual_kubelet_memory_request` (default: `128Mi`)
- `virtual_kubelet_memory_limit` (default: `512Mi`)

### Slurm defaults

- `slurm_partition` (default: `debug`)
- `slurm_default_cpu` (default: `1`)
- `slurm_default_memory` (default: `1G`)
- `slurm_job_time` (default: `00:10:00`)
- `slurm_extra_flags` (default: `[]`)
- `interlink_slurm_detect_capabilities` (default: `true`): derive node CPU, memory, and GPU metadata from Ansible facts gathered on a Slurm worker and export them as `slurm-capabilities.yml`.
- `interlink_slurm_capability_node` (default: empty): optional node name override written to the exported capability artifact.
- `interlink_slurm_capability_worker_host` (default: empty): explicit inventory host delegated for Slurm worker fact gathering.
- `interlink_slurm_capability_worker_group` (default: `slurm_workers`): inventory group used when `interlink_slurm_capability_worker_host` is empty; the first host in the group is used. If the group is absent, the role tries to infer a worker from inventory hosts or gathered nodenames containing `slurm-wn`.
- `interlink_slurm_capabilities_file` (default: `{{ interlink_artifact_dir }}/slurm-capabilities.yml`)
- `interlink_artifacts_wait_retries` (default: `60`): Kubernetes-side retries while waiting for Slurm-side mTLS artifacts.
- `interlink_artifacts_wait_delay` (default: `10`): seconds between Slurm-side artifact checks.
- `slurm_gpu_enabled` (default: `true`)
- `slurm_gpu_flavor` (default: `{{ slurm_partition }}-gpu`)
- `slurm_gpu_partition` (default: `{{ slurm_partition }}`)
- `slurm_gpu_count` (default: detected GPU count, falling back to `0`)
- `slurm_gpu_model` (default: detected GPU model, falling back to empty)
- `slurm_gpu_default_cpu` (default: detected Slurm worker CPUs, falling back to `slurm_default_cpu`)
- `slurm_gpu_default_memory` (default: detected Slurm worker memory in MB, falling back to `slurm_default_memory`)
- `slurm_gpu_gres` (default: generated from detected GPU model and count, for example `gpu:h100:1`)
- `slurm_gpu_extra_flags` (default: `[]`)

When `interlink_slurm_detect_capabilities` is enabled, the Slurm phase writes a local capability artifact derived from Ansible hardware facts gathered on a Slurm worker. The Kubernetes phase loads that artifact and advertises the detected CPU and memory as virtual-node capacity; if GPUs are detected in worker facts, it advertises `nvidia.com/gpu` accelerators and can add a GPU Slurm flavor with `--gres=gpu:<count>`. Memory is converted from MB to whole GiB for the Helm chart, rounded down to avoid overcommitting.

## Recommended Deployment Flow

Run the role in two phases:

1. Execute with `interlink_type_of_node: slurm` on your Slurm-side host.
2. Execute with `interlink_type_of_node: kubernetes` on your Kubernetes-side host.

The Kubernetes phase consumes artifacts exported by the Slurm phase from `interlink_artifact_dir`.

## Example Playbook: Slurm

```yaml
- name: Deploy InterLink on Slurm host
  hosts: slurm_frontend
  become: true
  roles:
    - role: grycap.interlink
      vars:
        interlink_type_of_node: slurm
        interlink_node_name: interlink-slurm-vk
```

## Example Playbook: Kubernetes

```yaml
- name: Deploy InterLink Virtual Kubelet on Kubernetes host
  hosts: k8s_frontend
  become: true
  roles:
    - role: grycap.interlink
      vars:
        interlink_type_of_node: kubernetes
        interlink_node_name: interlink-slurm-vk
        interlink_namespace: interlink
        interlink_run_smoke_test: true
```

## Dependency Role

`meta/main.yml` declares a conditional dependency:

- `usegalaxy_eu.apptainer` when running in Slurm mode.

## Troubleshooting

- Error about missing mTLS artifacts in Kubernetes mode:
  - Run Slurm mode first, then verify files exist in `interlink_artifact_dir`.
- CSR approval does not happen:
  - Verify `jq` is installed and the service account name matches `system:serviceaccount:<namespace>:<interlink_node_name>`.
- Helm install/upgrade fails:
  - Verify `helm` supports OCI and can access `ghcr.io`.

## License

Apache-2.0
