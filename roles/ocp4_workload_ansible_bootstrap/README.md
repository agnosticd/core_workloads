# ocp4_workload_ansible_bootstrap

Cluster-level Ansible bootstrap for multi-user OpenShift labs. Calls lab developer roles from a pre-installed Ansible collection to deploy and remove cluster-wide resources.

## Usage

Add to the cluster CI `workloads:` list. The lab developer's collection must be declared in `requirements_content` so it is installed before workloads run.

```yaml
requirements_content:
  collections:
  - name: https://github.com/acme/my-lab-content.git
    type: git
    version: main

workloads:
- agnosticd.core_workloads.ocp4_workload_ansible_bootstrap

remove_workloads:
- agnosticd.core_workloads.ocp4_workload_ansible_bootstrap

ocp4_workload_ansible_bootstrap_provision_roles:
- acme.my_lab.bootstrap_cluster

ocp4_workload_ansible_bootstrap_remove_roles:
- acme.my_lab.remove_cluster
```

## Variables

See `defaults/main.yml` for documented defaults.

## Lab developer collection conventions

The lab developer's collection should provide roles named in `provision_roles` and optionally `remove_roles`. Roles are called via `include_role` and have access to all standard agnosticd variables (`guid`, `openshift_api_url`, `openshift_cluster_ingress_domain`, etc.).

For tenant-level bootstrapping, use `ocp4_workload_tenant_ansible_bootstrap` in `namespaced_workloads` instead.
