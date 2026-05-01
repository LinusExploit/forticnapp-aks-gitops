# forticnapp-flux-gitops

GitOps repository reconciled by the Microsoft Flux extension on an AKS cluster.
Installs the Lacework / FortiCNAPP Kubernetes agent using the official Helm
chart, with the agent access token sourced from Azure Key Vault via the
Secrets Store CSI Driver.

This repository is intended to be pushed to a public GitHub repo. The Azure
infrastructure (AKS, Key Vault, RBAC, Flux extension, Flux configuration) is
provisioned separately by Terraform and is **not** stored here.

## Layout

```text
forticnapp-flux-gitops/
└── clusters/
    └── aks-cluster/
        ├── namespace.yaml
        ├── helmrepository.yaml
        ├── secretproviderclass.yaml
        ├── token-mounter.yaml
        ├── helmrelease.yaml
        └── kustomization.yaml
```

The Terraform `azurerm_kubernetes_flux_configuration` is configured to
reconcile from `./clusters/aks-cluster` on the `main` branch. Keep the path
consistent with the `flux_git_path` variable in the Terraform tfvars.

## What each file does

| File                            | Purpose                                                                                          |
| ------------------------------- | ------------------------------------------------------------------------------------------------ |
| `namespace.yaml`                | Creates the `lacework-agent` namespace.                                                          |
| `helmrepository.yaml`           | Flux `HelmRepository` pointing at `https://lacework.github.io/helm-charts/`.                     |
| `secretproviderclass.yaml`      | Tells the Secrets Store CSI Driver to fetch the token from Key Vault and sync it into a K8s Secret. |
| `token-mounter.yaml`            | Tiny pod that mounts the CSI volume so the synced Secret materializes (CSI requires a mount).    |
| `helmrelease.yaml`              | Installs the `lacework-agent` chart, reads the token from the synced Secret via `valuesFrom`.    |
| `kustomization.yaml`            | Plain `kustomize` index — Flux applies the resources listed here.                                |

## Placeholders to fill in before pushing

Run `terraform output` in the AKS Terraform folder to get the values you need.

**`secretproviderclass.yaml`**

| Placeholder         | Source                                                              |
| ------------------- | ------------------------------------------------------------------- |
| `<KV_NAME>`         | `key_vault_name` in `terraform.tfvars`                              |
| `<TENANT_ID>`       | `az account show --query tenantId -o tsv`                           |
| `<CSI_CLIENT_ID>`   | `terraform output -raw kv_secrets_provider_object_id` (this is the **client ID** of the addon identity) |

**`helmrelease.yaml`**

| Placeholder       | Suggested value                              |
| ----------------- | -------------------------------------------- |
| `<SERVER_URL>`    | Same as `lacework_server_url` in tfvars      |
| `<CLUSTER_NAME>`  | `aks-cluster` (or whatever `cluster_name` is)|
| `<ENV_NAME>`      | `dev` / `staging` / `prod` etc.              |

## Pushing to GitHub

```bash
cd "<this folder>"
git init -b main
git add .
git commit -m "Initial FortiCNAPP Flux manifests"
git remote add origin https://github.com/<your-org>/<your-repo>.git
git push -u origin main
```

Then set `flux_git_repository_url` in the Terraform tfvars to that URL and
run `terraform apply`.

## Quick sanity check after the agent installs

```bash
kubectl -n lacework-agent get helmrelease lacework-agent
kubectl -n lacework-agent get pods
kubectl -n lacework-agent logs -l app.kubernetes.io/name=lacework-agent --tail=50
```

Data takes 10–15 minutes to appear in the FortiCNAPP console after first install.

## References

- Lacework FortiCNAPP — Installing Linux agent on Kubernetes:
  https://docs.fortinet.com/document/lacework-forticnapp/latest/administration-guide/663510/installing-linux-agent-on-k8s
- Microsoft Flux extension for AKS:
  https://learn.microsoft.com/azure/azure-arc/kubernetes/conceptual-gitops-flux2
- Azure Key Vault Secrets Store CSI Driver on AKS:
  https://learn.microsoft.com/azure/aks/csi-secrets-store-driver
