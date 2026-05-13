# iag4-helm — Chart Reference for Claude

## What This Chart Deploys

**IAG (Itential Automation Gateway) version 4** — a Python-based network automation gateway.

- Chart name: `iag`, chart version: `1.0.4`, app version: `4.3.4`
- Deployed as a **StatefulSet** (1 replica — IAG is standalone, more than 1 is not supported)
- Image: `497639811223.dkr.ecr.us-east-2.amazonaws.com/automation-gateway` (ECR, requires imagePullSecret)
- Container entrypoint: `automation-gateway --sync-config`
- Listens on `applicationPort` (default `8443`, TLS by default)
- Uses **SQLite** for persistence — no MongoDB or Redis dependency
- Pod runs as UID/GID 1001, non-root

## Repository Layout

```
iag4-helm/
├── charts/iag4/
│   ├── Chart.yaml                  # name: iag, version: 1.0.4, appVersion: 4.3.4
│   ├── values.yaml                 # Full defaults with inline docs
│   ├── templates/
│   │   ├── _helpers.tpl            # iag.fullname, iag.labels, iag.selectorLabels, iag.annotations
│   │   ├── statefulset.yaml        # Core workload + init container (SSH key permissions)
│   │   ├── service.yaml            # ClusterIP service (port 443 → applicationPort)
│   │   ├── ingress.yaml            # Standard ingress (enabled by default)
│   │   ├── certificate.yaml        # cert-manager Certificate object
│   │   ├── issuer.yaml             # cert-manager Issuer (fails fast if name/caSecretName missing)
│   │   ├── storage-class.yaml      # StorageClass (optional)
│   │   ├── configmap.yaml          # ansible.cfg ConfigMap (when configMap.enabled)
│   │   └── configmap_ssh.yaml      # SSH client config (when configMap.enabled + files/ssh_config)
│   └── tests/
│       ├── test-values.yaml        # Shared values used by all test suites
│       ├── statefulset_test.yaml
│       ├── service_test.yaml
│       ├── ingress_test.yaml
│       ├── configmap_test.yaml
│       ├── configmap_ssh_test.yaml
│       ├── storage-class_test.yaml
│       ├── certificate_test.yaml
│       └── issuer_test.yaml
```

## Key Values & Defaults

| Value | Default | Notes |
|-------|---------|-------|
| `replicaCount` | `1` | Do not run more than 1 |
| `applicationPort` | `8443` | Container port IAG listens on |
| `useTLS` | `true` | Mounts TLS cert from secret, sets cert/key env vars |
| `image.repository` | (empty) | Must be set; ECR repo above for Itential-hosted |
| `image.tag` | (empty) | Must be set; falls back to `Chart.AppVersion` |
| `imagePullSecrets` | `[]` | Must be set for ECR |
| `serviceAccount.name` | (empty) | Set for IRSA/IAM bindings |
| `certManager.enabled` | `false` | Sub-chart off by default |
| `external-dns.enabled` | `false` | Sub-chart off by default |
| `storageClass.enabled` | `true` | Creates StorageClass object |
| `storageClass.provisioner` | `""` | e.g. `ebs.csi.aws.com` on AWS |
| `storageClass.reclaimPolicy` | `Retain` | Itential recommends Retain |
| `persistentVolumeClaims.dataClaim.storage` | `10Gi` | SQLite DBs + logs |
| `persistentVolumeClaims.codeClaim.storage` | `10Gi` | Customer scripts/Ansible/venvs |
| `configMap.enabled` | `true` | Mounts `ansible.cfg` into container |
| `resourcesEnabled` | `true` | Limits: 1000m/256Mi; Requests: 1000m/256Mi (low — override for prod) |
| `nodeSelector` | `{}` | No selector by default |
| `tolerations` | `{}` | No tolerations by default |

## Required Secrets (pre-existing, not created by chart)

| Secret name | When required | Keys used |
|------------|--------------|-----------|
| `<imagePullSecrets[0].name>` | Always | Docker ECR config |
| `<iag.fullname>-tls-secret` | `useTLS: true` | `tls.crt`, `tls.key`, `ca.crt` |
| `<issuer.caSecretName>` | `issuer.enabled: true` | CA cert for cert-manager Issuer |
| `<applicationSettings.hvSecretName>` | `hvEnabled: true` | `token`; + `ca.crt`, `tls.crt`, `tls.key` if `hvTLS: true` |
| `<applicationSettings.ldapSecretName>` | `ldapEnabled: true` | `password`; + `tls.crt` if `ldapTLSEnabled: true` |

`iag.fullname` = `<release-name>` if it contains the chart name `iag`, otherwise `<release-name>-iag`.

## Application Feature Toggles (`applicationSettings.*`)

All map to `automation_gateway_*` environment variables in the StatefulSet.

| Toggle | Default | Env var set |
|--------|---------|-------------|
| `ansibleEnabled` | `true` | `automation_gateway_ansible_enabled` |
| `ansibleVenvEnabled` | `true` | `automation_gateway_ansible_venv_enabled` (independent of `ansibleEnabled`) |
| `httpRequestsEnabled` | `true` | `automation_gateway_http_requests_enabled` |
| `netconfEnabled` | `true` | `automation_gateway_netconf_enabled` |
| `netmikoEnabled` | `true` | `automation_gateway_netmiko_enabled` |
| `nornirEnabled` | `false` | `automation_gateway_nornir_enabled` |
| `scriptsEnabled` | `true` | `automation_gateway_scripts_enabled` |
| `pythonVenvEnabled` | `true` | `automation_gateway_python_venv_enabled` |
| `grpcEnabled` | `true` | `automation_gateway_grpc_enabled` |
| `gitEnabled` | `true` | `automation_gateway_git_enabled` |
| `hvEnabled` | `false` | `automation_gateway_vault_enabled` |
| `ldapEnabled` | `false` | `automation_gateway_ldap_auth_enabled` |

Additional env vars can be injected freely via `applicationSettings.env` — the template iterates the map and sets every key/value as an env var. The full set of available IAG env vars is at https://docs.itential.com/docs/iag-configuration-2023-3.

## Fixed Environment Variables (not values-configurable)

These are hardcoded in the statefulset template:

| Env var | Value |
|---------|-------|
| `automation_gateway_global_log_directory` | `/var/lib/automation-gateway` |
| `automation_gateway_data_file` | `sqlite:////var/lib/automation-gateway/automation-gateway.db` |
| `automation_gateway_audit_db_file` | `sqlite:////var/lib/automation-gateway/automation-gateway_audit.db` |
| `automation_gateway_exec_history_db_file` | `sqlite:////var/lib/automation-gateway/automation-gateway_exec_history.db` |
| `automation_gateway_bind_address` | `status.podIP` (downward API) unless `applicationSettings.bindAddress` is set |
| `automation_gateway_terraform_enabled` | `"false"` (no values toggle) |

SQLite DB paths are fixed by the IAG application and must not be changed.

## Persistent Volumes

Two PVCs created via `volumeClaimTemplates` (when `persistentVolumeClaims.enabled: true`). Both use `storageClass.name`.

| PVC name | Mount path | Purpose |
|----------|-----------|---------|
| `iag-data-volume` | `/var/lib/automation-gateway` | SQLite DBs, audit DB, exec history DB, logs |
| `iag-code-volume` | `/usr/share/automation-gateway` | Customer assets (see structure below) |

Expected structure of `iag-code-volume`:
```
ansible/
  collections/
  inventory/hosts   ← inventory file path hardcoded in template
  modules/
  playbooks/
  roles/
  venv/
venvs/              ← python venv paths
scripts/
git/
  ssh/              ← id_rsa keys (init container sets 0400)
  repos/
nornir/
  config.yml
  modules/
  inventory/
```

## Init Container

`id-rsa-perms` runs before IAG starts. It scans `/usr/share/automation-gateway` for `id_rsa` files and `chmod 0400` each one. Required for SSH-based Git operations — SSH rejects keys with loose permissions. Uses the same image as the main container.

## Probes

All three probes are **disabled by default**. When enabled:

| Probe | Mechanism | Path/Command | Notes |
|-------|-----------|-------------|-------|
| Liveness | `exec` | `test $(pgrep -f automation-gateway \| wc -l) -ge 2` | Checks the process is running, not HTTP |
| Readiness | `httpGet` | `/api/v2.0/poll` on `applicationPort` | Scheme: HTTPS if `useTLS`, else HTTP |
| Startup | `httpGet` | `/api/v2.0/poll` on `applicationPort` | Max 3 min (18 × 10s); same scheme logic |

Liveness does **not** use httpGet — it uses `pgrep` to confirm at least 2 automation-gateway processes are running. This is different from readiness/startup.

## ConfigMaps

**`iag-config-map`** — created when `configMap.enabled: true`. Contains a hardcoded `ansible.cfg`:
```ini
[defaults]
collections_path=/usr/local/lib/python3.9/site-packages/ansible/collections:/usr/share/ansible/collections
```
Mounted at `/etc/ansible/ansible.cfg` (subPath mount).

**`ssh-config-map`** — created when `configMap.enabled: true` AND a file at `charts/iag4/files/ssh_config` exists in the chart directory at render time. The configmap is always created when `configMap.enabled` but only has data if the file is present. The volume mount in the statefulset is conditional on `.Files.Glob "files/ssh_config"`, so the mount is only added when the file exists. Mounted at `/etc/ssh/ssh_config`.

## TLS

When `useTLS: true`:
- Secret `<iag.fullname>-tls-secret` is mounted at `/etc/ssl/gateway` (read-only)
- Env vars `automation_gateway_server_certfile`, `keyfile`, `cabundle` point into that mount
- TLS terminates **at the pod** — the ingress passes traffic through, not terminated at LB by default
- For AWS ALB, use `alb.ingress.kubernetes.io/backend-protocol: HTTPS` and `scheme: internet-facing`

## Service & Ingress

**Service** — ClusterIP by default, named `iag-service`. External port 443 → `applicationPort` (8443).

**Ingress** — enabled by default. Ingress routes to `service.name` on `service.port` (443). No built-in TLS termination at ingress — ingress just proxies to the TLS-terminated pod.

## Optional Sub-charts

| Chart | Version | Condition |
|-------|---------|-----------|
| cert-manager | 1.12.3 | `certManager.enabled` |
| external-dns | 1.17.0 | `external-dns.enabled` |

cert-manager and external-dns are typically deployed as cluster-wide resources by infrastructure tooling (Tofu/Terraform/Ansible), not per-application. Keep both disabled unless you specifically need the chart to own them.

## cert-manager Integration

Two optional objects:

**Issuer** (`issuer.enabled: true`) — creates a cert-manager `Issuer` or `ClusterIssuer` backed by a CA secret. Fails at render time if `issuer.name` or `issuer.caSecretName` are empty.

**Certificate** (`certificate.enabled: true`) — creates a `Certificate` object that cert-manager fulfills into secret `<iag.fullname>-tls-secret`. References the issuer by `certificate.issuerRef.name` and `.kind`. Set `kind: ClusterIssuer` to reference a pre-existing cluster-scoped issuer.

## Common Helm Commands

```bash
# Install the helm-unittest plugin (one-time)
helm plugin install https://github.com/helm-unittest/helm-unittest

# Lint
helm lint charts/iag4

# Dry-run
helm install iag charts/iag4 -f my-values.yaml --dry-run --debug

# Install
helm install iag charts/iag4 -f my-values.yaml

# Upgrade
helm upgrade iag charts/iag4 -f my-values.yaml

# Run unit tests
helm unittest charts/iag4 --strict

# Regenerate snapshots after intentional template changes
helm unittest charts/iag4 --strict --update-snapshot
```

## Known Bugs & Gotchas

**`automation_gateway_terraform_enabled` is hardcoded `false`** — no values.yaml key controls this. Cannot be enabled via Helm.

**Default resource limits are very low** — `1000m CPU / 256Mi` is the default. IAG in production typically needs 2–4 CPU / 4–8Gi memory. Always override `resources` in production values files.
