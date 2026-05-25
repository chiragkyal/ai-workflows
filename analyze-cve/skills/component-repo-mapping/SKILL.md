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
3. **Jira component label** — look up the component name in the table below.
4. **Fuzzy match** — if no exact match, search the table for the closest partial match and confirm with the user.
5. **Prompt** — if still unresolved, ask the user for the repository URL.

---

## Component → Repository Map

### HyperShift / Hosted Control Planes

| Component / Label | GitHub Repository | Notes |
|---|---|---|
| HyperShift | `https://github.com/openshift/hypershift` | Main HyperShift operator |
| HyperShift / ROSA | `https://github.com/openshift/hypershift` | |
| hosted-control-planes | `https://github.com/openshift/hypershift` | |
| control-plane-operator | `https://github.com/openshift/hypershift` | Lives inside hypershift repo |
| hypershift-operator | `https://github.com/openshift/hypershift` | |

### OpenShift Control Plane

| Component / Label | GitHub Repository | Notes |
|---|---|---|
| kube-apiserver | `https://github.com/openshift/kubernetes` | OpenShift fork of k8s |
| kube-controller-manager | `https://github.com/openshift/kubernetes` | |
| openshift-apiserver | `https://github.com/openshift/openshift-apiserver` | |
| cluster-config-operator | `https://github.com/openshift/cluster-config-operator` | |
| cluster-kube-apiserver-operator | `https://github.com/openshift/cluster-kube-apiserver-operator` | |
| cluster-kube-controller-manager-operator | `https://github.com/openshift/cluster-kube-controller-manager-operator` | |
| cluster-kube-scheduler-operator | `https://github.com/openshift/cluster-kube-scheduler-operator` | |
| openshift-controller-manager | `https://github.com/openshift/openshift-controller-manager` | |

### Networking

| Component / Label | GitHub Repository | Notes |
|---|---|---|
| ovn-kubernetes | `https://github.com/ovn-org/ovn-kubernetes` | |
| cluster-network-operator | `https://github.com/openshift/cluster-network-operator` | |
| sdn | `https://github.com/openshift/sdn` | Legacy SDN |
| ingress | `https://github.com/openshift/router` | |
| cluster-ingress-operator | `https://github.com/openshift/cluster-ingress-operator` | |

### Storage

| Component / Label | GitHub Repository | Notes |
|---|---|---|
| csi-driver | `https://github.com/openshift/csi-driver-shared-resource` | |
| cluster-storage-operator | `https://github.com/openshift/cluster-storage-operator` | |
| aws-ebs-csi-driver-operator | `https://github.com/openshift/aws-ebs-csi-driver-operator` | |

### Authentication / Security

| Component / Label | GitHub Repository | Notes |
|---|---|---|
| oauth-server | `https://github.com/openshift/oauth-server` | |
| oauth-apiserver | `https://github.com/openshift/oauth-apiserver` | |
| cluster-authentication-operator | `https://github.com/openshift/cluster-authentication-operator` | |

### Cloud Providers

| Component / Label | GitHub Repository | Notes |
|---|---|---|
| GCP / HCP on GKE | `https://github.com/openshift/hypershift` | GCP platform code lives in hypershift |
| AWS / ROSA | `https://github.com/openshift/hypershift` | |
| Azure | `https://github.com/openshift/hypershift` | |
| cluster-api | `https://github.com/openshift/cluster-api` | |
| cluster-api-provider-aws | `https://github.com/openshift/cluster-api-provider-aws` | |
| cluster-api-provider-azure | `https://github.com/openshift/cluster-api-provider-azure` | |
| cluster-api-provider-gcp | `https://github.com/openshift/cluster-api-provider-gcp` | |

### Builds / CI

| Component / Label | GitHub Repository | Notes |
|---|---|---|
| openshift-builds | `https://github.com/openshift/builds` | |
| shipwright | `https://github.com/shipwright-io/build` | |
| tekton | `https://github.com/tektoncd/pipeline` | |

### Common / Shared Libraries

| Component / Label | GitHub Repository | Notes |
|---|---|---|
| library-go | `https://github.com/openshift/library-go` | |
| api | `https://github.com/openshift/api` | OpenShift API types |
| client-go | `https://github.com/openshift/client-go` | |

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
