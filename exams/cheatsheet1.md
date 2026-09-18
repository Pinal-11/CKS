# CKS Last-Minute Revision Notes

*Quick-scan cheat sheet — grouped by domain, distilled from the 15-question mock exam.*

---

## ⏱️ Exam-Day Reflexes

- **Always `kubectl config use-context <name>`** first — wrong cluster = zero points on that question.
- Static pods (`kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `etcd`) live in `/etc/kubernetes/manifests/` — **editing = auto-restart** by kubelet. No `kubectl apply` needed/possible.
- Before touching `kube-apiserver.yaml`, know there's usually a **backup** provided — note its path immediately in case you break the API server.
- `allowPrivilegeEscalation` and `readOnlyRootFilesystem` → **container-level** securityContext only.
- `runAsUser`, `runAsNonRoot`, `seccompProfile`, `appArmorProfile` → can be **pod-level** (inherited) or container-level (override).
- Admission controllers (PSA, ImagePolicyWebhook, etc.) only check **new/incoming** requests — never retroactively affect already-running pods.

---

## 🔒 AppArmor

```bash
apparmor_parser -q /path/to/profile        # load profile
aa-status | grep <profile-name>            # verify loaded
```

Pod spec:

```yaml
securityContext:
  appArmorProfile:
    type: Localhost
    localhostProfile: <profile-name>   # matches "profile <name> {" inside the file
```

- Profile name ≠ filename — always open the file to check the actual `profile NAME {` declaration.
- Must load on the **node** the pod will run on.

---

## 🧩 Seccomp

- Default profiles root: **`/var/lib/kubelet/seccomp/`**
- Custom profiles conventionally go in a `profiles/` subfolder under that root.

```yaml
securityContext:
  seccompProfile:
    type: Localhost
    localhostProfile: profiles/<file>.json   # path relative to seccomp root
```

- `type: RuntimeDefault` = use container runtime's default seccomp (no file needed) — this is what PSA's `restricted` level demands if you don't supply a custom profile.

---

## 🔑 Secrets Handling

| Concern | Rule of thumb |
|---|---|
| Extract & decode | `kubectl get secret <s> -o jsonpath='{.data.<key>}' \| base64 -d` |
| Preferred consumption | **Volume mount > env var** (env vars leak via `/proc/<pid>/environ`, crash dumps, child processes, `kubectl describe`) |
| Read-only mount | Always set `readOnly: true` on the volumeMount |
| Static analysis red flags | hard-coded creds in `ENV`/`ARG`, secrets in earlier Docker layers even after `RUN rm`, `.env`/`.git` copied into build context, kubeconfig baked into image |

```bash
docker history --no-trunc <image>   # inspect layers for leaked build-time secrets
```

---

## 📜 SBOM (Software Bill of Materials)

```bash
bom generate --image-archive <tarball>.tar --format json --output <out>.spdx
```

- Workflow: find the container → `apk info | grep <pkg>` to identify runtime packages → `describe pod` for the exact image name → match tarball by name → generate SBOM.
- SPDX = one of the two standard SBOM formats (the other being CycloneDX).

---

## 👤 ServiceAccounts & RBAC

**Least privilege checklist:**
- Never use `default` SA for workloads that need real permissions.
- Compare Roles/RoleBindings across SAs to find the minimally-scoped one.
- Delete unused SAs once migration is done.

**Disable automount:**

```yaml
# On the ServiceAccount
automountServiceAccountToken: false
```

```yaml
# Also set on the Pod/Deployment spec for defense-in-depth
spec:
  automountServiceAccountToken: false
```

**Manually mount a token anyway (projected volume):**

```yaml
volumes:
  - name: sa-token
    projected:
      sources:
        - serviceAccountToken:
            path: <name>
            expirationSeconds: 3600
            audience: default
```

**RBAC scoping pattern:**

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]        # narrow verbs = least privilege
```

- Same object **names** can be reused across namespaces; re-create (delete + apply) if a Role is over-permissioned — don't just create a new name.

---

## 🛡️ Pod Security Admission (PSA)

Check enforcement label:

```bash
kubectl get ns <ns> --show-labels
# pod-security.kubernetes.io/enforce=restricted|baseline|privileged
```

**`restricted` profile requires (per container, unless noted):**

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  runAsNonRoot: true          # pod OR container level
  seccompProfile:
    type: RuntimeDefault      # pod OR container level
# and: do NOT set runAsUser: 0 anywhere
```

**Diagnosis flow:**

```bash
kubectl describe deployment <d> -n <ns>     # Conditions: ReplicaFailure
kubectl describe replicaset <rs> -n <ns>    # Events: "forbidden ... violates PodSecurity"
```

---

## 🌐 NetworkPolicy

```bash
kubectl get pods -n <ns> --show-labels
kubectl get ns <ns> --show-labels           # kubernetes.io/metadata.name=<ns> is always present
```

**AND vs OR logic — the #1 exam trap:**

```yaml
ingress:
- from:
  - namespaceSelector: {...}     # \
    podSelector: {...}           #  } same list item = AND (both must match)
  - namespaceSelector: {...}     # new dash = OR (separate condition)
```

- No `NetworkPolicy` in a namespace = **all traffic allowed** by default.
- Once **any** policy selects a pod for `Ingress`, all non-matching ingress traffic to that pod is denied (default-deny becomes implicit for that direction).
- `podSelector: {}` (empty) = selects **all pods** in the namespace.

---

## 🕵️ Falco

- Default rules: `/etc/falco/falco_rules.yaml` — **never edit directly**.
- Custom/override rules: `/etc/falco/falco_rules.local.yaml` — a rule with the **same `rule:` name** overrides the original.
- Enable file output in `/etc/falco/falco.yaml`:

```yaml
file_output:
  enabled: true
  keep_alive: false
  filename: /path/to/alerts.log
```

- Reload without full restart:

```bash
kill -1 $(cat /var/run/falco.pid)
# or: systemctl restart falco
```

- Priority levels (low→high): `Debug, Informational, Notice, Warning, Error, Critical, Emergency`.
- Give new log output up to ~60s to start appearing.

---

## 📋 Audit Logging

**Policy rule ordering = top-down, first match wins.** Put specific rules before catch-alls.

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages: ["RequestReceived"]
rules:
  - level: Metadata
    verbs: ["delete"]
    resources: [{group: "", resources: ["secrets"]}]
    namespaces: ["kube-system"]
  - level: Request
    verbs: ["create","update","patch","delete"]
    resources: [{group: "apps", resources: ["deployments"]}]
    namespaces: ["default"]
  - level: Metadata      # catch-all, must be last
```

**Audit levels:** `None < Metadata < Request < RequestResponse`

**kube-apiserver flags:**

```
--audit-policy-file=<path>
--audit-log-path=<path>
--audit-log-maxage=<days>
--audit-log-maxbackup=<count>
--audit-log-maxsize=<MB>
```

Don't forget matching `hostPath` volumes + `volumeMounts` for both the policy file and the log directory.

---

## 🚦 Admission Controllers (ImagePolicyWebhook)

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: <path-to-kubeconfig>
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false      # false = fail CLOSED (reject on webhook failure) — safer default
```

```
--admission-control-config-file=<path>
--enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
```

- Existing running pods are **not** retroactively checked — must `get -o yaml`, edit, `delete`, `apply` to force re-evaluation.
- Always strip `resourceVersion`, `uid`, `creationTimestamp`, `status` before re-applying an exported pod YAML.

---

## 🌍 Ingress + TLS

```yaml
spec:
  ingressClassName: nginx
  tls:
    - hosts: [<hostname>]
      secretName: <tls-secret>     # must already exist, type kubernetes.io/tls
  rules:
    - host: <hostname>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: <svc>
                port:
                  number: <port>
```

```bash
kubectl get svc -n ingress-nginx                 # find controller external IP
echo "<IP> <hostname>" | sudo tee -a /etc/hosts
curl -k https://<hostname>                        # -k to skip self-signed cert check
```

---

## 🎯 One-Line Reminders Before You Submit Each Question

- Did I use the **least-privileged** option available (SA, role, capability set) rather than the easiest one?
- Did I edit the **override/local** file instead of the default config where one exists?
- Did I **delete + recreate** anything that admission/PSA/policy changes require to take effect?
- Did I leave **unrelated fields untouched** when the task said "don't change X"?
- Did I verify with a **describe/status/log check**, not just assume the apply succeeded?
- Is the **namespace** correct in every manifest and command?