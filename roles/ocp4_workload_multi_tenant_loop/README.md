# ocp4_workload_multi_tenant_loop

Generic bridge workload that provisions tenant workloads for N users on a shared OpenShift cluster.

Designed for **combined catalog items** that merge cluster-level and tenant-level provisioning into a single order. This role sits at the end of the `workloads:` list and loops a separate `tenant_workloads:` list once per user.

## How it works

```
workloads:                              # run once by openshift-workloads config
├── ocp4_workload_authentication
├── ocp4_workload_pipelines
├── ...cluster workloads...
└── ocp4_workload_multi_tenant_loop     # this role
        │
        │  reads: num_users, tenant_workloads
        │  loops: user1 → userN
        │
        ├── tenant_keycloak_user  (user1)
        ├── tenant_namespace      (user1)
        ├── tenant_showroom       (user1)
        ├── tenant_keycloak_user  (user2)
        ├── ...
        └── tenant_showroom       (userN)
```

For each user, the role overrides two variables via `include_role vars:`:

| Variable | Value |
|---|---|
| `ocp4_workload_tenant_keycloak_username` | `{{ tenant_user_prefix }}{{ user_number }}` |
| `common_password` | Auto-generated per user |

All other tenant variables in the catalog item should be Jinja2 expressions that derive from these two roots. Ansible evaluates Jinja2 lazily, so overriding the root causes the entire variable tree to resolve correctly for each user.

## Required variables

Set these in the AgnosticV catalog item:

| Variable | Type | Description |
|---|---|---|
| `num_users` | int | Number of tenant users to provision |
| `tenant_workloads` | list | Workload FQCNs to run per user (provision order) |
| `tenant_remove_workloads` | list | Workload FQCNs for destroy (explicit order) |

## Optional variables

| Variable | Default | Description |
|---|---|---|
| `tenant_user_prefix` | `user` | Username prefix (users: user1, user2, ...) |
| `tenant_user_offset` | `1` | Starting user number |
| `tenant_password_length` | `8` | Per-user password length |

## Example: AgnosticV catalog item

```yaml
config: openshift-workloads
num_users: 16

workloads:
- agnosticd.core_workloads.ocp4_workload_authentication
- agnosticd.core_workloads.ocp4_workload_pipelines
# ...cluster workloads...
- agnosticd.core_workloads.ocp4_workload_multi_tenant_loop

tenant_workloads:
- agnosticd.namespaced_workloads.ocp4_workload_tenant_keycloak_user
- agnosticd.namespaced_workloads.ocp4_workload_tenant_namespace
- agnosticd.showroom.ocp4_workload_showroom

tenant_remove_workloads:
- agnosticd.showroom.ocp4_workload_showroom
- agnosticd.namespaced_workloads.ocp4_workload_tenant_namespace
- agnosticd.namespaced_workloads.ocp4_workload_tenant_keycloak_user

remove_workloads:
- agnosticd.core_workloads.ocp4_workload_multi_tenant_loop
- agnosticd.core_workloads.ocp4_workload_pipelines
- agnosticd.core_workloads.ocp4_workload_authentication

# Tenant vars — all derive from ocp4_workload_tenant_keycloak_username
ocp4_workload_tenant_namespace_username: >-
  {{ ocp4_workload_tenant_keycloak_username }}
ocp4_workload_showroom_namespace: >-
  {{ ocp4_workload_tenant_keycloak_username }}-showroom
```

## Destroy behavior

On destroy, the config's `remove_workloads` calls this role first. The role reads `tenant_remove_workloads` and loops users in reverse order (last user destroyed first). After all tenants are cleaned up, the config continues destroying cluster-level workloads.

## Passwords

Each user gets a unique password written to `{{ output_dir }}/tenant_user_{{ N }}_password`. The Ansible `password` lookup plugin generates on first call and returns the same value on re-runs — safe for idempotent provisioning.

## Limitations

- Provisioning is sequential (user1 completes all workloads before user2 starts)
- If a user fails, subsequent users are not provisioned
- No parallel provisioning in v1 — can be added as a follow-up if needed
