# k3s-apps-ansible

Ansible Galaxy collection `roncemer.k3s_apps` with roles and playbooks to install:

- `cloudnativepg`: the [CloudNativePG operator Helm chart](https://github.com/cloudnative-pg/charts).
- `wordpress`: the [Helmforge WordPress Helm chart](https://github.com/helmforgedev/charts/tree/main/charts/wordpress).

CloudNativePG installs the operator, not a PostgreSQL database cluster. WordPress
uses its chart's bundled MySQL by default; it does not use CloudNativePG.

## Prerequisites

- Ansible-core 2.16+ with the `kubernetes.core` collection (installed as a collection dependency).
- An existing k3s cluster and a host with Helm 3+, Python, and PyYAML installed.
- An existing `k3s` Unix user (or the user selected by `kube_api_user`) on that host,
  with a working kubeconfig and sufficient Kubernetes permissions to install the
  charts, including CloudNativePG CRDs and cluster-scoped resources.
- The Ansible connection user must be able to become `kube_api_user` via sudo.
  Unprivileged-to-unprivileged escalation may require POSIX ACL support (`acl`).
- For WordPress's default persistent volumes, a default StorageClass (such as
  k3s's local-path provisioner).

The roles do not install Helm, create users, or change kubeconfig permissions.

## Install and run

From this repository:

```bash
ansible-galaxy collection build --output-path /tmp
ansible-galaxy collection install /tmp/roncemer-k3s_apps-1.0.0.tar.gz
ansible-playbook -i inventory.ini roncemer.k3s_apps.cloudnativepg
ansible-playbook -i inventory.ini roncemer.k3s_apps.wordpress
```

Your inventory must contain a `k3s` group. Target **one** API-capable host per
cluster so multiple hosts do not concurrently manage the same Helm release:

```ini
[k3s]
k3s-server ansible_host=192.0.2.10
```

All Helm operations, including repository management, run with `become: true`
and `become_user: "{{ kube_api_user }}"`. The roles need no raw kubectl commands;
run any manual kubectl commands as the same user.

## Configuration

Set variables in inventory, playbook variables, or with `--extra-vars`.

| Variable | Default | Purpose |
| --- | --- | --- |
| `kube_api_user` | `k3s` | Unix user for all Helm/Kubernetes operations |
| `kubeconfig` | `""` | Optional kubeconfig path on the target host; empty uses the selected user's Helm defaults |
| `cloudnativepg_chart_repo_url` | `https://cloudnative-pg.github.io/charts` | CloudNativePG chart repository |
| `cloudnativepg_chart_version` | `""` | Optional pinned chart version; empty selects latest |
| `cloudnativepg_release_name` | `cnpg` | Helm release name |
| `cloudnativepg_namespace` | `cnpg-system` | Namespace, created if absent |
| `cloudnativepg_values` | `{}` | CloudNativePG chart values |
| `cloudnativepg_wait_timeout` | `10m` | Helm readiness timeout |
| `wordpress_chart_repo_url` | `https://repo.helmforge.dev` | Helmforge chart repository |
| `wordpress_chart_version` | `""` | Optional pinned chart version; empty selects latest |
| `wordpress_release_name` | `wordpress` | Helm release name |
| `wordpress_namespace` | `wordpress` | Namespace, created if absent |
| `wordpress_values` | `{}` | Helmforge WordPress chart values |
| `wordpress_wait_timeout` | `10m` | Helm readiness timeout |

For example, choose a different API user and an explicit k3s kubeconfig:

```bash
ansible-playbook -i inventory.ini roncemer.k3s_apps.cloudnativepg \
  -e kube_api_user=cluster-admin -e kubeconfig=/etc/rancher/k3s/k3s.yaml
```

That user must already have permission to read the specified kubeconfig. Both
roles can also be used in your own plays by their fully qualified names:
`roncemer.k3s_apps.cloudnativepg` and `roncemer.k3s_apps.wordpress`.

Pass chart-specific options (storage, ingress, resources, database configuration)
through the corresponding `*_values` dictionary. Consult the upstream chart
documentation linked above. WordPress defaults are for development/evaluation:
configure TLS, backups, resources, and credentials for production. Prefer
existing Kubernetes Secrets or Ansible Vault rather than plaintext passwords.
WordPress installation output is suppressed to avoid exposing sensitive chart
values. Re-running either role manages the existing Helm release and waits for
readiness; pin chart versions for reproducible deployments.
