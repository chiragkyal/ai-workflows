---
name: component-repo-mapping
description: Resolve a Red Hat/OpenShift component name or Jira component label to one or more GitHub repository URLs for cloning
---

# Component to Repository Mapping

Translates a component name (from a Jira ticket, user input, or CVE advisory) into one or more GitHub repository URLs that should be cloned and analyzed.

## When to Use This Skill

Use this skill when:
- `--repo` was not provided explicitly
- A Jira ticket was provided and its `components` field contains recognisable Red Hat/OpenShift component names
- The CVE advisory mentions a specific Go module path and you need to find its source repository
- The user provides a short component alias instead of a full GitHub URL

---

## Resolution Order

Try these steps in order and stop at the first hit:

1. **Exact `--repo` URL** — user already gave a full `https://github.com/...` URL → use it directly, skip this skill.
2. **Go module path** — if CVE profile contains an affected package like `github.com/org/repo`, derive the clone URL directly.
3. **Jira `components` field** — look up exact component names in the table below.
4. **Jira summary + description scan** — search the ticket summary and description text for any keyword or image name listed in the "Aliases / Image Names" column of the table below.
5. **Fuzzy match** — if no exact match, find the closest partial match and confirm with the user before proceeding.
6. **Prompt** — if still unresolved, ask the user for the repository URL.

---

## Component → Repository Map

The table uses three columns:
- **Component / Label** — exact Jira component field value or `--repo` alias
- **Aliases / Image Names** — additional keywords scanned in ticket summary and description
- **GitHub Repository / Repositories**

### Certificates

| Component / Label | Aliases / Image Names | GitHub Repository |
|---|---|---|
| cert-manager | cert-manager, cert-manager-operator, ose-cert-manager | `https://github.com/openshift/cert-manager-operator` |

### Secrets / CSI

| Component / Label | Aliases / Image Names | GitHub Repository |
|---|---|---|
| secrets-store-csi-driver-operator | secrets-store-csi-driver-operator | `https://github.com/openshift/secrets-store-csi-driver-operator` |
| secrets-store-csi-driver | secrets-store-csi-driver, secrets-store-csi | `https://github.com/openshift/secrets-store-csi-driver` |

### Workload Identity

| Component / Label | Aliases / Image Names | GitHub Repository |
|---|---|---|
| zero-trust-workload-identity-manager | zero-trust-workload-identity-manager, spire, spire-operator | `https://github.com/openshift/spire-operator` |

### Operator Framework / SDK

| Component / Label | Aliases / Image Names | GitHub Repository |
|---|---|---|
| Operator SDK | Operator SDK, operator-sdk, operator sdk, ose-ansible, ocp-release-operator-sdk | `https://github.com/openshift/ocp-release-operator-sdk` |
| ansible-operator-plugins | ansible-operator, ose-ansible-operator, ansible-operator-plugins | `https://github.com/openshift/ansible-operator-plugins` |

### External Secrets

| Component / Label | Aliases / Image Names | GitHub Repository |
|---|---|---|
| external-secrets-operator | external-secrets-operator, External Secrets Operator, ose-external-secrets-operator | `https://github.com/openshift/external-secrets-operator` |

### Diagnostics

| Component / Label | Aliases / Image Names | GitHub Repository |
|---|---|---|
| must-gather | must-gather, ose-must-gather | `https://github.com/openshift/must-gather` |
| must-gather-operator | must-gather-operator | `https://github.com/openshift/must-gather-operator` |

---

## Go Module Path → Repository

If the CVE's affected package is a Go module path, derive the repository directly:

```
github.com/openshift/hypershift          → https://github.com/openshift/hypershift
github.com/openshift/library-go          → https://github.com/openshift/library-go
golang.org/x/net                         → https://github.com/golang/net
golang.org/x/crypto                      → https://github.com/golang/crypto
golang.org/x/text                        → https://github.com/golang/text
k8s.io/apiserver                         → https://github.com/kubernetes/apiserver
k8s.io/client-go                         → https://github.com/kubernetes/client-go
k8s.io/api                               → https://github.com/kubernetes/api
sigs.k8s.io/controller-runtime          → https://github.com/kubernetes-sigs/controller-runtime
```

For any `github.com/<org>/<repo>` path, the clone URL is `https://github.com/<org>/<repo>`.

---

## Multiple Repositories

A Jira ticket may list multiple components. In that case:
- Resolve ALL components to their repositories.
- If there are 3 or fewer, clone all of them and run the analysis against each.
- If there are more than 3, list them and ask the user which to prioritize.

---

## Return Value

Return to parent workflow:

```json
{
  "skill": "component-repo-mapping",
  "status": "success",
  "resolved_repos": [
    {
      "component": "HyperShift / ROSA",
      "repo_url": "https://github.com/openshift/hypershift",
      "clone_path": "/workspace/repos/hypershift",
      "confidence": "exact_match"
    }
  ],
  "unresolved": [],
  "resolution_method": "jira_component_label"
}
```

`confidence` values: `exact_match`, `fuzzy_match`, `module_path`, `user_provided`

---

## Integration with Parent Workflow

Called from **Phase 0.7** of the CVE Analysis workflow (see `CLAUDE.md`).

**Input:** `--repo` value OR `jira_context.components` from Phase 0.5 OR affected package from Phase 1
**Output:** List of `(component, repo_url, clone_path)` tuples used in Phase 0.7 to clone repos
