# CKS Exam Recall — Reconstructed Practice Questions

Based on your post-exam recall. Reconstructed at CKS difficulty with full solutions and exam-pattern notes for each.

---

## Q1 — Identify Image with Package Installed + Generate SBOM

**Task (reconstructed):**
```
Several Alpine-based images are running in the cluster (or provided as tarballs).
Identify which image has a specific package (e.g. curl, openssl) installed.
Generate an SBOM (SPDX format) for that identified image.
Save the SBOM to ~/image.spdx and the image/container name to ~/identified.txt.
```

**Solution:**
```bash
# Check each container for the package
kubectl exec -n <ns> <pod> -c <container1> -- apk info | grep <package>
kubectl exec -n <ns> <pod> -c <container2> -- apk info | grep <package>

# Once found, note the image name
kubectl describe pod <pod> -n <ns>   # find image under the matching container

# Generate SBOM (from tarball, if provided)
bom generate --image-archive /root/ImageTarballs/<image>.tar \
  --format json --output ~/image.spdx

# Or directly from a live/registry image if no tarball given
bom generate --image <image>:<tag> --format json --output ~/image.spdx

echo "<container-name>" > ~/identified.txt
```

**Exam trap:** `apk info` only works on Alpine-based images (uses `apk`, not `apt`/`yum`). If a container errors out on `apk info`, that's actually useful information too — it means that container likely isn't Alpine-based and can be ruled out quickly rather than debugged.

---

## Q2 — Remove User from Group Inside a Running System, Restart Container

**Task (reconstructed):**
```
A container is running with a user `fdevelop` that has unnecessary membership
in the `daemon` group. Remove fdevelop from the daemon group,
then restart the affected container so the change takes effect.
```

**Solution:**
```bash
# Identify the container/pod
kubectl get pods -n <ns>

# Exec in and check current groups
kubectl exec -it <pod> -n <ns> -- id fdevelop
kubectl exec -it <pod> -n <ns> -- cat /etc/group | grep daemon

# Remove fdevelop from the daemon group
kubectl exec -it <pod> -n <ns> -- gpasswd -d fdevelop daemon
# or, if gpasswd isn't available in the minimal image:
kubectl exec -it <pod> -n <ns> -- deluser fdevelop daemon    # Alpine/busybox style
```

**Restart the container** — a group membership change inside a running container's filesystem is usually **ephemeral** unless it's baked into the image or a persistent volume, so if the task wants this to persist, the fix likely needs to happen at the **image/Dockerfile level**, not just live inside the running container. But if the task literally just asks you to fix the running state and restart:
```bash
kubectl delete pod <pod> -n <ns>
# if managed by a Deployment, it recreates automatically with the ORIGINAL image
# (meaning your live fix would be lost unless the underlying image was also fixed)
```

**Important distinction to check on the real task wording:** if restarting via `kubectl delete pod` would just respawn the same misconfigured user setup (because it's baked into the image), the actual fix belongs in the **Dockerfile** (remove the `usermod -aG daemon fdevelop` line or equivalent), rebuild, and redeploy — not a live in-container patch that gets wiped on restart. Read the task carefully for which layer it's actually testing.

---

## Q3 — Layered Audit Policy (Namespace + Resource + Catch-all)

**Task (reconstructed):**
```
Update the audit policy to:
1. Log RequestResponse level for all requests in a specific namespace (e.g. finance)
2. Log Request level for the "web-app" Deployment in namespace "prod"
3. Log Metadata level for everything else
```

**Solution:**
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  - level: RequestResponse
    namespaces: ["finance"]

  - level: Request
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "apps"
        resources: ["deployments"]
        resourceNames: ["web-app"]
    namespaces: ["prod"]

  - level: Metadata
```
Apply to the API server manifest exactly as in previous audit tasks:
```yaml
- --audit-policy-file=/etc/kubernetes/<policy-file>.yaml
- --audit-log-path=/var/log/<path>.log
```
Wait for kubelet to auto-restart the static pod, verify with `ls -l` on the log path and `kubectl get pods -n kube-system`.

**Exam trap:** rule **order matters** — the most specific rules must come before the general catch-all `level: Metadata` rule (no filters), since Kubernetes audit policy evaluates rules top-to-bottom and uses the **first match**.

---

## Q4 — Istio Strict Mutual TLS for a Namespace

**Task (reconstructed):**
```
Enable Istio sidecar injection for Namespace X (if not already enabled).
Enforce STRICT mutual TLS for all workloads in that namespace using a PeerAuthentication resource.
```

**Solution:**
```bash
kubectl label namespace <ns> istio-injection=enabled
kubectl rollout restart deployment -n <ns>   # for existing pods to get the sidecar
```
```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: <ns>
spec:
  mtls:
    mode: STRICT
```
```bash
kubectl apply -f peerauth.yaml
```

**Verify:**
```bash
kubectl get pods -n <ns>   # confirm 2/2 containers (istio-proxy present)
istioctl authn tls-check <pod>.<ns>
```

**Exam trap:** `PeerAuthentication` named `default` in a specific namespace overrides any mesh-wide default policy for that namespace only — naming it `default` is the convention Istio expects for the namespace-scoped baseline policy, not arbitrary.

---

## Q5 — ServiceAccount Automount False + Projected Volume Token

*(Same pattern as your earlier practice exam Q8 — reconstructed again here for completeness since it recurred.)*

**Solution:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <sa-name>
  namespace: <ns>
automountServiceAccountToken: false
```
```yaml
spec:
  template:
    spec:
      serviceAccountName: <sa-name>
      automountServiceAccountToken: false
      volumes:
        - name: sa-token
          projected:
            sources:
              - serviceAccountToken:
                  path: token
                  expirationSeconds: 3600
      containers:
        - name: <container>
          volumeMounts:
            - name: sa-token
              mountPath: /var/run/secrets/tokens
              readOnly: true
```

---

## Q6 — Cilium Ingress with TLS + SSL Redirect

**Task (reconstructed):**
```
Create an Ingress using the Cilium ingress class for service "web" (port 80),
hostname "web.example.com", with TLS termination via an existing secret,
and HTTP-to-HTTPS redirect enabled.
```

**Solution:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: <ns>
  annotations:
    ingress.cilium.io/ssl-redirect: "true"
spec:
  ingressClassName: cilium
  tls:
    - hosts:
        - web.example.com
      secretName: web-tls
  rules:
    - host: web.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```
```bash
kubectl apply -f web-ingress.yaml
kubectl get ingress -n <ns>
```

**Exam trap:** Cilium's Ingress annotation prefix is `ingress.cilium.io/...`, **not** `nginx.ingress.kubernetes.io/...` — using the nginx-style annotation on a Cilium-backed Ingress silently does nothing since Cilium's controller doesn't recognize it. Always confirm `ingressClassName` matches what's actually installed in the cluster:
```bash
kubectl get ingressclass
```

---

## Q7 — CIS Benchmark Remediation via kubeadm-managed Config

**Task (reconstructed):**
```
kube-bench has flagged a specific CIS control (e.g. related to a kubeadm-managed
component config). Fix it via the kubeadm ConfigMap/manifest mechanism so the
fix persists across kubeadm-driven reconfigurations, not just a one-off manual edit.
```

**Solution — general pattern (same as your KubeletConfiguration task earlier):**
```bash
kube-bench run --targets=master,node \
  --config-dir=/opt/kube-bench/cfg \
  --benchmark=cis-1.10 \
  --config=/opt/kube-bench/cfg/config.yaml
```
Identify the specific FAIL, then apply the fix at the correct layer:
- If it's a **kubelet** setting → edit the `kubelet-config` ConfigMap in `kube-system`, then `kubeadm upgrade node phase kubelet-config` on each node.
- If it's a **static pod flag** (API server, controller-manager, scheduler, etcd) → edit the manifest under `/etc/kubernetes/manifests/`.
- If it's a **ClusterConfiguration**-level setting → `kubectl edit cm kubeadm-config -n kube-system`, though this typically only takes effect on the next `kubeadm upgrade`/`init` — some settings still require a direct manifest edit to take effect immediately.

Re-run kube-bench to confirm PASS.

---

## Q8 — Pod Security Admission "Restricted" Compliance Fix

*(Matches the KodeKloud practice paper Q9 pattern you referenced.)*

**Task:**
```
Deployment is failing to schedule in a namespace enforcing the "restricted" PSA level.
Diagnose and fix without changing the namespace labels or image.
```

**Solution:**
```bash
kubectl describe replicaset -n <ns>   # Events show exact PSA rejection reasons
```
```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
  # remove any runAsUser: 0
```

---

## Q9 — ImagePolicyWebhook (variant)

*(Matches killer.sh CKS1 Q12 pattern, "slightly different" per your note — likely different paths/webhook service name.)*

**Solution (general pattern):**
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: <container-side mounted path to kubeconfig>
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false
```
```yaml
# kube-apiserver.yaml additions
- --admission-control-config-file=<container-side mounted path to admission-configuration.yaml>
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
```

**Critical reminder from your actual exam troubleshooting earlier this week:** always use the **container-mounted path** in both the flag and the `kubeConfigFile` reference, never the host path — this exact mistake caused a real crash-loop in your practice session. Double-check `volumeMounts` in the manifest for the correct in-container path before writing the flag value.

Remediate any already-existing non-compliant pod (admission controllers don't retroactively affect running pods):
```bash
kubectl get pod <pod> -n <ns> -o yaml > fix.yaml
# edit image tag away from :latest
kubectl delete pod <pod> -n <ns>
kubectl apply -f fix.yaml
```

---

## Q10 — kubeadm Minor Version Upgrade (Worker Node Only)

**Task (reconstructed):**
```
Upgrade the worker node to the next minor version using kubeadm and apt.
```

**Solution:**
```bash
# On the worker node
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=<version>-1.1
apt-mark hold kubeadm

kubeadm upgrade node          # NOT "upgrade apply" — that's control-plane only

apt-mark unhold kubelet kubectl
apt-get install -y kubelet=<version>-1.1 kubectl=<version>-1.1
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet
```
From the control plane:
```bash
kubectl drain <worker> --ignore-daemonsets --delete-emptydir-data   # before upgrading
kubectl uncordon <worker>                                            # after
```

**Exam trap:** `kubeadm upgrade apply` is control-plane-only — running it on a worker node fails. Workers always use `kubeadm upgrade node`.

---

## Q11 — Dockerfile + Deployment Security Best Practices (Two-File Fix)

**Task (reconstructed):**
```
A Dockerfile and a deployment.yaml are provided with security anti-patterns.
Fix the USER in the Dockerfile so the container doesn't run as root.
Fix the deployment.yaml's securityContext (readOnlyRootFilesystem, runAsNonRoot) to follow best practice.
```

**Solution — Dockerfile:**
```dockerfile
FROM node:20-alpine

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY . .
RUN chown -R appuser:appgroup /app

USER appuser   # was previously missing, or set to root/UID 0

CMD ["node", "server.js"]
```

**Solution — deployment.yaml:**
```yaml
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
```

**Exam trap — this is the one you flagged yourself:** watch for a deliberately-planted *wrong-direction* change like `readOnlyRootFilesystem: false` or a missing `USER` line — the task is testing whether you can spot an anti-pattern that was **intentionally weakened**, not just whether you can write a hardened config from a blank file. Read the existing file carefully before editing — don't just paste in a "best practice" template blindly, since the existing values (image name, working dir, app-specific env vars) still need to be preserved exactly as-is aside from the specific security fields being fixed.

---

## Q12 — NetworkPolicy: Deny-All Ingress + Cross-Namespace Allow

**Task (reconstructed):**
```
1. Create a deny-all-ingress NetworkPolicy in a namespace (e.g. db)
2. Create a second policy allowing ingress ONLY from the "prod" namespace into "db"
```

**Solution:**
```yaml
# 1. Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: db
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress: []
---
# 2. Allow only from prod namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-prod
  namespace: db
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: prod
```
```bash
kubectl apply -f deny-all-ingress.yaml -f allow-from-prod.yaml
```

**Exam trap:** an empty `ingress: []` with `podSelector: {}` denies **all** ingress traffic to **every** pod in the namespace — this is the correct default-deny baseline. The second policy then **adds** to what's allowed (NetworkPolicies are additive/union when multiple apply to the same pod) — you don't need to merge both rules into a single policy object; two separate policies achieve the same combined effect.

---

## Q13 — Create a TLS Secret from Cert + Key (Don't Touch the Deployment)

**Task (reconstructed):**
```
Given a certificate file and key file, create a TLS Secret with the required name,
in the required namespace, WITHOUT modifying the existing Deployment that will consume it.
```

**Solution:**
```bash
kubectl create secret tls <secret-name> \
  --cert=/path/to/tls.crt \
  --key=/path/to/tls.key \
  -n <namespace>
```

**Verify:**
```bash
kubectl get secret <secret-name> -n <namespace>
kubectl describe secret <secret-name> -n <namespace>
# type should show: kubernetes.io/tls
```

**Exam trap:** the instruction "do not make any changes in the Deployment" is almost always testing whether you assume you also need to wire up a `volumeMount`/`env` reference to the new secret — **don't**. If the Deployment isn't explicitly asked to be modified, it likely already references this secret name (e.g., an Ingress `tls.secretName` or a pre-existing volume reference expecting this exact secret to now exist) — your only job is getting the Secret's name, namespace, and content exactly right so whatever already references it starts working. Adding unrequested changes to the Deployment risks breaking the grading check that specifically verifies the Deployment was left untouched.

---

## Cross-cutting reminders from this batch
- **Container-mounted paths vs host paths** — this tripped you up for real on the ImagePolicyWebhook task this week; it's clearly a recurring exam pattern (admission config, kubeconfig references, volume mounts) — always trace the actual `volumeMounts.mountPath`, never assume the host path works inside the container.
- **"Don't modify X"** instructions are almost always a hint about what NOT to over-engineer, not an arbitrary constraint — respecting them precisely is itself part of the grading.
- **Layer-matching fixes** (kubeadm ConfigMap vs static manifest vs live in-container patch) — several of these questions are really testing whether you fix the problem at the *correct, persistent* layer rather than a fix that looks right but gets silently overwritten or reset later.