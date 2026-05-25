---
name: image-repo-mapping
description: Map a container image name (from a Jira ticket summary, pscomponent label, or Downstream Component Name field) to the GitHub repository that should be cloned for CVE impact analysis
---

# Image to Repository Mapping

Translates a container image name into the GitHub repository URL to clone for analysis. Image names appear in the ticket summary, `pscomponent:` labels, and the `Downstream Component Name` custom field.

## When to Use This Skill

Use this skill when:
- An image name has been extracted from a Jira ticket (summary, label, or custom field)
- The user passes `--repo=<image-name>` using a short image name rather than a full GitHub URL
- Phase 0.7 needs to determine which repository to clone

---

## Resolution Order

Apply these in order and stop at the first match:

1. **Full GitHub URL supplied** (`--repo=https://github.com/...`) → use directly, skip this skill.
2. **Exact image name match** — look the image name up in the table below.
3. **Prefix match** — strip version suffixes (e.g. `-1-0`, `-1-12-4`, `-rhel9`, `-rhel8`) and re-match.
4. **Namespace prefix match** — if image is `<namespace>/<name>`, match on the `<namespace>` alone.
5. **Fuzzy match** — find the closest entry and confirm with the user.
6. **Prompt** — ask the user for the GitHub URL.

> **Tip:** Strip `pkg:oci/` prefix before matching (e.g. `pkg:oci/ose-ansible-operator` → `ose-ansible-operator`).

---

## Image → Repository Map

> Every image name here has appeared in real ticket summaries or labels.

### cert-manager / jetstack

The `cert-manager` namespace spans **three** repositories — match on the full image name, not just the namespace prefix.

#### cert-manager-operator (operator + bundle only)

| Image Name / Prefix | GitHub Repository |
|---|---|
| `cert-manager-operator-container` | `https://github.com/openshift/cert-manager-operator` |
| `cert-manager-operator-rhel9` | `https://github.com/openshift/cert-manager-operator` |
| `cert-manager/cert-manager-operator-rhel9` | `https://github.com/openshift/cert-manager-operator` |
| `cert-manager/cert-manager-operator-bundle` | `https://github.com/openshift/cert-manager-operator` |

#### cert-manager-istio-csr

| Image Name / Prefix | GitHub Repository |
|---|---|
| `cert-manager/cert-manager-istio-csr-rhel9` | `https://github.com/openshift/cert-manager-istio-csr` |

#### jetstack-cert-manager

| Image Name / Prefix | GitHub Repository |
|---|---|
| `cert-manager/jetstack-cert-manager-rhel9` | `https://github.com/openshift/jetstack-cert-manager` |
| `cert-manager/jetstack-cert-manager-acmesolver-rhel9` | `https://github.com/openshift/jetstack-cert-manager` |
| `jetstack-cert-manager-rhel9` | `https://github.com/openshift/jetstack-cert-manager` |
| `jetstack-cert-manager-container` | `https://github.com/openshift/jetstack-cert-manager` |
| `jetstack-cert-manager-acmesolver-rhel9` | `https://github.com/openshift/jetstack-cert-manager` |
| `jetstack-cert-manager-acmesolver-container` | `https://github.com/openshift/jetstack-cert-manager` |
| `redhat-user-workloads/jetstack-cert-manager-*` | `https://github.com/openshift/jetstack-cert-manager` |

**Namespace prefix match:** `cert-manager` → requires full image name lookup above (multiple repos in this namespace).  
**Keyword match:** `cert-manager-operator` / `cert-manager-operator-bundle` → `cert-manager-operator`; `cert-manager-istio-csr` → `cert-manager-istio-csr`; `jetstack-cert-manager` → `jetstack-cert-manager`.

---

### Operator SDK / Helm Operator (non-ansible images)

| Image Name / Prefix | GitHub Repository |
|---|---|
| `operator-sdk` | `https://github.com/openshift/ocp-release-operator-sdk` |
| `openshift4/ose-operator-sdk-rhel8` | `https://github.com/openshift/ocp-release-operator-sdk` |
| `openshift4/ose-operator-sdk-rhel9` | `https://github.com/openshift/ocp-release-operator-sdk` |
| `openshift-enterprise-operator-sdk-container` | `https://github.com/openshift/ocp-release-operator-sdk` |
| `openshift4/ose-helm-operator` | `https://github.com/openshift/ocp-release-operator-sdk` |
| `openshift4/ose-helm-rhel9-operator` | `https://github.com/openshift/ocp-release-operator-sdk` |
| `openshift-enterprise-helm-operator-container` | `https://github.com/openshift/ocp-release-operator-sdk` |
| `pkg:oci/ose-operator-sdk-rhel8` | `https://github.com/openshift/ocp-release-operator-sdk` |
| `pkg:oci/ose-operator-sdk-rhel9` | `https://github.com/openshift/ocp-release-operator-sdk` |
| `pkg:oci/ose-helm-operator` | `https://github.com/openshift/ocp-release-operator-sdk` |
| `pkg:oci/openshift-enterprise-helm-operator` | `https://github.com/openshift/ocp-release-operator-sdk` |

---

### Ansible Operator

| Image Name / Prefix | GitHub Repository |
|---|---|
| `ansible-operator-plugins` | `https://github.com/openshift/ansible-operator-plugins` |
| `openshift4/ose-ansible-operator` | `https://github.com/openshift/ansible-operator-plugins` |
| `openshift4/ose-ansible-rhel9-operator` | `https://github.com/openshift/ansible-operator-plugins` |
| `openshift-enterprise-ansible-operator-container` | `https://github.com/openshift/ansible-operator-plugins` |
| `pkg:oci/ose-ansible-operator` | `https://github.com/openshift/ansible-operator-plugins` |
| `pkg:oci/ose-ansible-rhel9-operator` | `https://github.com/openshift/ansible-operator-plugins` |

**Keyword match:** any image containing `ansible-operator` → `https://github.com/openshift/ansible-operator-plugins`

---

### External Secrets Operator

The `external-secrets-operator` namespace spans **three** repositories — match on the full image name.

#### external-secrets-operator (operator + bundle)

| Image Name / Prefix | GitHub Repository |
|---|---|
| `external-secrets-operator/external-secrets-operator-rhel9` | `https://github.com/openshift/external-secrets-operator` |
| `external-secrets-operator/external-secrets-operator-bundle` | `https://github.com/openshift/external-secrets-operator` |
| `redhat-user-workloads/external-secrets-operator-1-0` | `https://github.com/openshift/external-secrets-operator` |
| `redhat-user-workloads/external-secrets-operator-bundle-1-0` | `https://github.com/openshift/external-secrets-operator` |

#### external-secrets

| Image Name / Prefix | GitHub Repository |
|---|---|
| `external-secrets-operator/external-secrets-rhel9` | `https://github.com/openshift/external-secrets` |
| `redhat-user-workloads/external-secrets-1-0` | `https://github.com/openshift/external-secrets` |

#### external-secrets-bitwarden-sdk-server

| Image Name / Prefix | GitHub Repository |
|---|---|
| `external-secrets-operator/bitwarden-sdk-server-rhel9` | `https://github.com/openshift/external-secrets-bitwarden-sdk-server` |
| `redhat-user-workloads/bitwarden-sdk-server-1-0` | `https://github.com/openshift/external-secrets-bitwarden-sdk-server` |

**Namespace prefix match:** `external-secrets-operator` → requires full image name lookup above (multiple repos in this namespace).  
**Keyword match:** `external-secrets-operator` / `external-secrets-operator-bundle` → `external-secrets-operator`; `bitwarden-sdk-server` → `external-secrets-bitwarden-sdk-server`; `external-secrets-rhel9` / `external-secrets-1-0` → `external-secrets`.

---

### Zero Trust / SPIFFE / SPIRE

The `zero-trust-workload-identity-manager` namespace spans **four** repositories — match on the full image name, not just the namespace prefix.

#### spire-operator (operator + bundle only)

| Image Name / Prefix | GitHub Repository |
|---|---|
| `zero-trust-workload-identity-manager/zero-trust-workload-identity-manager-rhel9` | `https://github.com/openshift/spire-operator` |
| `zero-trust-workload-identity-manager/zero-trust-workload-identity-manager-operator-bundle` | `https://github.com/openshift/spire-operator` |

#### spiffe-spire (agent, server, OIDC discovery provider)

| Image Name / Prefix | GitHub Repository |
|---|---|
| `zero-trust-workload-identity-manager/spiffe-spire-agent-rhel9` | `https://github.com/openshift/spiffe-spire` |
| `zero-trust-workload-identity-manager/spiffe-spire-server-rhel9` | `https://github.com/openshift/spiffe-spire` |
| `zero-trust-workload-identity-manager/spiffe-spire-oidc-discovery-provider-rhel9` | `https://github.com/openshift/spiffe-spire` |
| `redhat-user-workloads/spiffe-spire-agent-*` | `https://github.com/openshift/spiffe-spire` |
| `redhat-user-workloads/spiffe-spire-server-*` | `https://github.com/openshift/spiffe-spire` |
| `redhat-user-workloads/spiffe-spire-oidc-discovery-provider-*` | `https://github.com/openshift/spiffe-spire` |

#### spiffe-spire-controller-manager

| Image Name / Prefix | GitHub Repository |
|---|---|
| `zero-trust-workload-identity-manager/spiffe-spire-controller-manager-rhel9` | `https://github.com/openshift/spiffe-spire-controller-manager` |

#### spiffe-spiffe-csi (CSI driver)

| Image Name / Prefix | GitHub Repository |
|---|---|
| `zero-trust-workload-identity-manager/spiffe-csi-driver-rhel9` | `https://github.com/openshift/spiffe-spiffe-csi` |
| `redhat-user-workloads/spiffe-csi-driver-*` | `https://github.com/openshift/spiffe-spiffe-csi` |

#### spiffe-spiffe-helper

| Image Name / Prefix | GitHub Repository |
|---|---|
| `zero-trust-workload-identity-manager/spiffe-helper-rhel9` | `https://github.com/openshift/spiffe-spiffe-helper` |

**Namespace prefix match:** `zero-trust-workload-identity-manager` → requires full image name lookup above (multiple repos in this namespace).  
**Keyword match:** `spire-agent` / `spire-server` / `spire-oidc` → `https://github.com/openshift/spiffe-spire`; `spire-controller-manager` → `https://github.com/openshift/spiffe-spire-controller-manager`; `spiffe-csi-driver` → `https://github.com/openshift/spiffe-spiffe-csi`; `spiffe-helper` → `https://github.com/openshift/spiffe-spiffe-helper`; `zero-trust-workload-identity-manager` (manager/bundle) → `https://github.com/openshift/spire-operator`.

---

### must-gather

| Image Name / Prefix | GitHub Repository |
|---|---|
| `openshift4/ose-must-gather-rhel9` | `https://github.com/openshift/must-gather` |
| `openshift4/ose-support-log-gather-rhel9-operator` | `https://github.com/openshift/must-gather-operator` |

---

### Secrets Store CSI _(no historical tickets yet — mappings pre-registered)_

| Image Name / Prefix | GitHub Repository |
|---|---|
| `secrets-store-csi-driver-operator` | `https://github.com/openshift/secrets-store-csi-driver-operator` |
| `secrets-store-csi-driver` | `https://github.com/openshift/secrets-store-csi-driver` |

---

## Return Value

```json
{
  "skill": "image-repo-mapping",
  "status": "success",
  "resolved_repos": [
    {
      "image_name": "openshift4/ose-ansible-rhel9-operator",
      "repo_url": "https://github.com/openshift/ansible-operator-plugins",
      "clone_path": "/workspace/repos/ansible-operator-plugins",
      "confidence": "exact_match",
      "match_method": "exact"
    }
  ],
  "unresolved": [],
  "resolution_method": "summary_image_name"
}
```

`confidence`: `exact_match`, `prefix_match`, `namespace_match`, `keyword_match`, `user_provided`

---

## Integration with Parent Workflow

Called from **Phase 0.7** of the CVE Analysis workflow (see `CLAUDE.md`).

**Input:** image name extracted by `jira-cve-extraction` skill (from summary, `pscomponent:` label, or `Downstream Component Name` field)  
**Output:** `(image_name, repo_url, clone_path)` tuples passed to Phase 0.7 for cloning
