# CKS Exam Tools Cheatsheet

All tools are assumed pre-installed on the exam cluster unless the task explicitly says "install."
Focus: fast invocation, reading output, and the follow-up fix — not memorizing every flag.

---

## 1. kube-bench — CIS Benchmark scanning
**What it does:** Checks node/cluster config (API server, kubelet, etcd flags, file permissions) against the CIS Kubernetes Benchmark.

```bash
# Run all checks on current node role (auto-detects master/node)
kube-bench run

# Run only a specific target
kube-bench run --targets master
kube-bench run --targets node
kube-bench run --targets etcd

# Run a single check by ID
kube-bench run --targets master --check 1.2.1

# Save output
kube-bench run --targets master > /root/kube-bench-report.txt
```
**Pattern:** find `[FAIL]` lines → the "Remediation" text tells you exactly which flag/file to fix → edit the static pod manifest (`/etc/kubernetes/manifests/kube-apiserver.yaml` etc.) → kubelet auto-restarts it.

---

## 2. Trivy — vulnerability scanning
**What it does:** Scans container images, filesystems, and k8s manifests for CVEs, secrets, misconfig.

```bash
# Scan an image, all severities
trivy image nginx:1.16

# Only HIGH/CRITICAL (most common exam ask)
trivy image --severity HIGH,CRITICAL nginx:1.16

# Ignore unfixed vulns (no patch available yet)
trivy image --severity HIGH,CRITICAL --ignore-unfixed nginx:1.16

# Output to file
trivy image --severity HIGH,CRITICAL nginx:1.16 > /root/report.txt

# Scan a Dockerfile / filesystem for misconfig
trivy fs --scanners vuln,secret,config /path/to/dir

# Scan a live k8s cluster/namespace
trivy k8s --report summary cluster
trivy k8s -n <namespace> all
```
**Pattern:** scan → identify the specific CVE or count asked for → often followed by "update the deployment to use a patched image tag" or "remove the offending image."

---

## 3. Falco — runtime threat detection
**What it does:** Watches syscalls in real time, alerts on suspicious behavior (shell in container, unexpected file writes, etc.) via rules.

```bash
# Check Falco is running
systemctl status falco          # if run as systemd service
kubectl get pods -n falco       # if run as daemonset

# View live alerts
falco                           # foreground
journalctl -fu falco            # if systemd
tail -f /var/log/falco.log      # if logging to file (check falco.yaml for path)

# Rules file location (default)
/etc/falco/falco_rules.local.yaml    # put custom rules HERE, not in falco_rules.yaml

# Reload after editing rules
systemctl restart falco
# or, for a pod-based Falco:
kubectl delete pod -n falco -l app=falco   # let daemonset recreate it
```
**Custom rule example:**
```yaml
- rule: Terminal shell in container
  desc: A shell was spawned inside a container
  condition: spawned_process and container and shell_procs
  output: Shell spawned (user=%user.name container=%container.name)
  priority: WARNING
```
**Pattern:** write/verify a rule → trigger the behavior (e.g., `kubectl exec -it <pod> -- sh`) → confirm the alert fires in the log.

---

## 4. OPA (standalone, not Gatekeeper) — policy engine
**What it does:** Validates/denies API requests via admission webhook, policies written in Rego.

```bash
# Load a rego policy into OPA (via kube-mgmt sidecar + labeled ConfigMap)
kubectl create configmap samplepolicy \
  --from-file=/root/sample.rego -n opa \
  --dry-run=client -o yaml | \
  kubectl label --local -f - openpolicyagent.org/policy=rego -o yaml | \
  kubectl apply -f -

# Verify it loaded (check annotation added by kube-mgmt)
kubectl get configmap samplepolicy -n opa -o yaml
# look for: openpolicyagent.org/policy-status: '{"status":"ok"}'
```
**Pattern:** name in the task = ConfigMap name; label `openpolicyagent.org/policy=rego` is mandatory or nothing loads; `--from-file` auto-keys the data by filename.

---

## 5. Kubesec — static manifest analysis
**What it does:** Scores a pod/deployment YAML for security risk (privileged, missing limits, hostNetwork, etc.)

```bash
# Local binary
kubesec scan pod.yaml

# Via docker (common if not installed locally)
docker run -i kubesec/kubesec:v2 scan /dev/stdin < pod.yaml
```
**Pattern:** run scan → read the "critical"/"advise" score comments → fix the manifest (e.g., add `readOnlyRootFilesystem: true`, drop capabilities, remove `privileged: true`).

---

## 6. KubeLinter — static analysis (policy-as-code)
**What it does:** Lints k8s YAML against best-practice checks (similar goal to kubesec, different tool).

```bash
kube-linter lint pod.yaml
kube-linter lint deployment.yaml --config kube-linter-config.yaml
```

---

## 7. conftest / OPA (policy testing on files)
**What it does:** Tests YAML/JSON files against Rego policies outside the cluster (CI-style).

```bash
conftest test deployment.yaml -p policy/
```

---

## 8. Syft — SBOM generation
**What it does:** Generates a Software Bill of Materials for an image or filesystem.

```bash
syft nginx:1.16 -o table
syft nginx:1.16 -o cyclonedx-json > /root/sbom.json
syft nginx:1.16 -o spdx-json > /root/sbom-spdx.json
```
**Also via Trivy** (single-tool alternative):
```bash
trivy image --format cyclonedx --output /root/sbom.json nginx:1.16
```

---

## 9. cosign — image signing/verification
**What it does:** Signs and verifies container image signatures (supply chain integrity).

```bash
cosign sign --key cosign.key <registry>/<image>:<tag>
cosign verify --key cosign.pub <registry>/<image>:<tag>
```

---

## 10. AppArmor — mandatory access control (System Hardening)
```bash
# Check status / loaded profiles
apparmor_status

# Load a profile
apparmor_parser -q /etc/apparmor.d/<profile-file>

# Set profile to enforce/complain mode
aa-enforce /etc/apparmor.d/<profile-file>
aa-complain /etc/apparmor.d/<profile-file>
```
**Apply to a pod:**
```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/<container-name>: localhost/<profile-name>
```

---

## 11. Seccomp — syscall filtering (System Hardening)
**Profile location on nodes:** `/var/lib/kubelet/seccomp/profiles/`

```yaml
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/<profile-name>.json
  # or, to use runtime default:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
```
**Verify:**
```bash
kubectl exec <pod> -- cat /proc/1/status | grep Seccomp
```

---

## 12. gVisor / Kata Containers — runtime sandboxing
**What it does:** Runs containers in an isolated sandbox/VM instead of sharing the host kernel directly.

```bash
# Check available RuntimeClasses
kubectl get runtimeclass
```
```yaml
# runtimeclass.yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
---
# Pod using it
spec:
  runtimeClassName: gvisor
```

---

## 13. Pod Security Standards / Admission (PSS/PSA)
**What it does:** Namespace-level enforcement of Baseline/Restricted/Privileged security profiles — replaces old PodSecurityPolicy.

```bash
kubectl label namespace <ns> \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```
**Debug a rejected pod:**
```bash
kubectl describe pod <pod> -n <ns>
# look for admission-webhook rejection reason in Events
```

---

## 14. Audit Logging
**Config lives in the API server static pod manifest:**
```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-log-path=/var/log/kubernetes/audit/audit.log
- --audit-log-maxage=30
- --audit-log-maxbackup=10
```
**Policy levels:** `None` / `Metadata` / `Request` / `RequestResponse`
```bash
# Verify logs are being written
tail -f /var/log/kubernetes/audit/audit.log
```

---

## 15. RBAC verification
```bash
kubectl auth can-i <verb> <resource> --as=<user-or-sa> -n <ns>
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>
```

---

## 16. NetworkPolicy debugging
```bash
kubectl describe networkpolicy <name> -n <ns>
kubectl get pods -n <ns> --show-labels
kubectl get endpoints <svc> -n <ns>
```

---

## 17. Docker/containerd hardening checks
```bash
# Docker socket ownership (should be root:root)
ls -l /var/run/docker.sock

# Daemon not listening on TCP
cat /etc/docker/daemon.json

# Confirm user not in docker group
groups <username>
```

---

## Fast reference table

| Tool | Domain | One-line purpose |
|---|---|---|
| kube-bench | Cluster/System Hardening | CIS benchmark checks |
| Trivy | Supply Chain | Image/fs vulnerability + SBOM scanning |
| Falco | Monitoring/Runtime | Runtime syscall-based threat detection |
| OPA | Cluster Hardening | Admission-time policy enforcement (Rego) |
| Kubesec | Supply Chain | Static YAML risk scoring |
| KubeLinter | Supply Chain | Static YAML best-practice linting |
| Syft | Supply Chain | SBOM generation |
| cosign | Supply Chain | Image signing/verification |
| AppArmor | System Hardening | MAC profiles on syscalls/file access |
| Seccomp | System Hardening | Syscall filtering per container |
| gVisor/Kata | Microservice Vuln. | Sandboxed container runtime |
| PSS/PSA | Microservice Vuln. | Namespace-level pod security enforcement |
| Audit logging | Monitoring | Records all API server requests |

---

### Exam-day reminders
- Tools are pre-installed unless the task explicitly says "install" (and even then, expect a local package, not internet download).
- Fix only what's asked — don't "clean up" extra findings, it costs time for zero credit.
- After any config change to a static pod (API server, kubelet), the kubelet watches `/etc/kubernetes/manifests/` and auto-restarts it — no manual restart needed, but wait for it to come back before verifying.
- Always verify with `describe`/`get events`/`logs` after every fix — a "looks right" YAML that silently fails is the most common way to lose points.