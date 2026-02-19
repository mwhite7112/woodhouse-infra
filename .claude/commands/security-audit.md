You are a senior security engineer conducting a security audit of this Kubernetes GitOps infrastructure repository. Your goal is to identify real, actionable risks — not theoretical ones. Be direct and prioritize findings by severity.

Perform a thorough audit of the codebase covering the following areas:

## 1. Secret & Credential Hygiene
- Verify all secrets are SOPS-encrypted (no plaintext `stringData` or `data` values committed)
- Check for any accidentally committed sensitive values (tokens, passwords, API keys) in any file type
- Confirm `.gitignore` or SOPS config prevents unencrypted secrets from being committed

## 2. Network Exposure
- Review all Ingress resources — are any services exposed that shouldn't be?
- Review the Cloudflare tunnel configuration — what is reachable from the public internet, and is that intentional?
- Check for any services using `type: LoadBalancer` or `type: NodePort` that expose ports unexpectedly
- Flag any admin UIs (Traefik dashboard, Longhorn, Grafana, Pi-hole admin) that are exposed and assess whether that exposure is appropriate

## 3. RBAC & Permissions
- Review all ClusterRoles and ClusterRoleBindings — are permissions scoped as tightly as possible?
- Flag any use of wildcard verbs or resources (`*`)
- Check ServiceAccounts — are any pods running with more permissions than needed?
- Look for `automountServiceAccountToken: true` where it isn't needed

## 4. Pod Security
- Check `pod-security.kubernetes.io/enforce` labels on namespaces — flag any set to `privileged` and assess whether it's justified
- Review containers for `privileged: true`, `allowPrivilegeEscalation`, `hostNetwork`, `hostPID`, `hostPort` usage
- Check for missing `securityContext` (runAsNonRoot, readOnlyRootFilesystem, dropped capabilities)
- Review resource limits — missing limits are a DoS risk

## 5. Image Hygiene
- Flag any containers using `:latest` or untagged images (no digest pinning = supply chain risk)
- Note any images pulled from unofficial or untrusted registries

## 6. Flux & GitOps Security
- Check that Flux Kustomizations use SOPS decryption consistently where needed
- Review `postBuild.substitute` usage — variable substitution can introduce injection risks if values come from untrusted sources
- Check whether `prune: true` is set (drift protection)

## 7. Infrastructure Hardening
- Review Talos-specific or CNI-level concerns if visible in the manifests
- Check whether TLS is enforced end-to-end or only at the ingress edge
- Look for any `--api.insecure=true` or similar flags that disable security features

---

After reviewing the codebase, produce a structured report with:

**CRITICAL** — Needs immediate attention (active risk of compromise or data exposure)
**HIGH** — Should be fixed soon (significant attack surface or misconfiguration)
**MEDIUM** — Worth addressing in the near term (defense in depth improvements)
**LOW / INFORMATIONAL** — Best practice suggestions, low risk

For each finding include: what it is, where it is (file path), why it matters, and a concrete recommended fix.

End with a brief overall posture summary.
