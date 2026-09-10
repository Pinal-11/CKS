# CKS Reference Cheat Sheet: Linux User/Group Management & System Hardening

## Part 1: Linux User, Group & Password Commands

### User Management

```bash
# Create a user (default: creates home dir, adds to /etc/passwd, /etc/shadow, /etc/group)
useradd username

# Create user with specific options (common in exam scenarios)
useradd -m -d /home/username -s /bin/bash -c "comment" username
# -m : create home directory
# -d : specify home directory path
# -s : specify login shell
# -c : comment/full name

# Create user with a specific UID
useradd -u 1500 username

# Set/change password for a user
passwd username
# then enter new password twice (interactive)

# Set password non-interactively (useful in scripts)
echo "username:password" | chpasswd

# Modify an existing user
usermod -l newname oldname        # rename login
usermod -d /new/home -m username  # change home dir (and move contents)
usermod -s /bin/sh username       # change shell
usermod -aG groupname username    # ADD to a supplementary group (-a = append, IMPORTANT: without -a it replaces all groups)
usermod -L username               # lock account (disable password login)
usermod -U username               # unlock account

# Delete a user
userdel username
userdel -r username     # also remove home directory and mail spool

# Check user info
id username
id -Gn username     # show group names user belongs to
```

### Group Management

```bash
# Create a group
groupadd groupname

# Create group with specific GID
groupadd -g 2000 groupname

# Add existing user to a group (secondary/supplementary)
usermod -aG groupname username
# OR
gpasswd -a username groupname

# Remove user from a group
gpasswd -d username groupname

# Delete a group
groupdel groupname

# Change group name/GID
groupmod -n newname oldname
groupmod -g newgid groupname

# List members of a group
getent group groupname
```

### Password / Account Info

```bash
# View password aging/expiry info
chage -l username

# Force password expiry (user must change at next login)
chage -d 0 username

# Set account expiry date
chage -E 2026-12-31 username

# Check password file entries directly
cat /etc/passwd | grep username
cat /etc/shadow | grep username     # requires root
getent passwd username
```

### Key Files Reference

| File | Purpose |
|---|---|
| `/etc/passwd` | user account info (uid, gid, home, shell) |
| `/etc/shadow` | encrypted passwords, expiry |
| `/etc/group` | group definitions |
| `/etc/gshadow` | secure group info |
| `/etc/login.defs` | default settings (UID/GID ranges etc.) |
| `/etc/skel/` | template files copied to new home dirs |

---

## Part 2: System Hardening (AppArmor, Seccomp, Capabilities)

### AppArmor

```bash
# Check if AppArmor is enabled on the node
cat /sys/module/apparmor/parameters/enabled
aa-status                    # list loaded profiles + their mode

# Load a custom profile from a file
apparmor_parser /etc/apparmor.d/custom-profile
# or reload after edits:
apparmor_parser -r /etc/apparmor.d/custom-profile

# List all currently loaded profiles
apparmor_parser -QT /etc/apparmor.d/*    # quiet test-load
aa-status                                # shows enforce/complain mode per profile

# Put a profile in complain mode (logs violations, doesn't block) — useful to generate a profile
aa-complain /etc/apparmor.d/custom-profile

# Put a profile in enforce mode (actively blocks)
aa-enforce /etc/apparmor.d/custom-profile

# Check kernel/dmesg for AppArmor denials (great for debugging exam tasks)
dmesg | grep -i apparmor
journalctl -k | grep -i apparmor
```

**Applying it to a Pod** (native field, GA since k8s v1.30 — no longer needs the old
`container.apparmor.security.beta.kubernetes.io/<container>` annotation on newer clusters,
but check the cluster version in the exam):

```yaml
spec:
  securityContext:
    appArmorProfile:
      type: Localhost
      localhostProfile: nginx-profile   # must already be loaded on the node
  containers:
  - name: web
    image: nginx
```

Older/annotation-based style (still valid, and used if targeting <1.30):

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/web: localhost/nginx-profile
```

### Seccomp

```bash
# Check if seccomp is supported/enabled on node kernel
grep -i seccomp /boot/config-$(uname -r)

# Default location kubelet looks for local seccomp profiles
ls /var/lib/kubelet/seccomp/profiles/

# Copy a custom seccomp JSON profile into place (path relative to profiles dir)
cp my-profile.json /var/lib/kubelet/seccomp/profiles/

# Trace which syscalls a running process makes (to build a custom profile)
strace -c -f -p <PID>       # count syscalls used by a live process
strace -f ./mybinary        # trace a fresh run
```

**Applying it to a Pod:**

```yaml
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault      # or: Localhost / Unconfined
      # localhostProfile: profiles/audit.json   # only if type: Localhost
  containers:
  - name: app
    image: myapp
```

Verify inside the container:

```bash
kubectl exec -it <pod> -- cat /proc/1/status | grep Seccomp
# Seccomp: 2  → 0=disabled, 1=strict, 2=filtered
```

### Capabilities & Privilege Restriction

```yaml
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
```

```bash
# Check capabilities of a running process on the node
getpcaps <PID>
capsh --print
```

### Kernel Modules / sysctls (host hardening)

```bash
# List loaded kernel modules
lsmod

# Blacklist an unneeded module (hardening step)
echo "install <module_name> /bin/false" >> /etc/modprobe.d/blacklist.conf

# View/set sysctl kernel params (e.g., disabling core dumps, ip forwarding checks)
sysctl -a | grep kernel
sysctl -w kernel.dmesg_restrict=1
```

### Audit & Verification (CIS Benchmark tooling)

```bash
# Run kube-bench (CIS benchmark checker) — common CKS task
kube-bench run --targets node
kube-bench run --targets master
./kube-bench --config-dir `pwd`/cfg --config `pwd`/cfg/config.yaml

# Check file permissions/ownership on critical files (common CIS finding)
stat -c "%a %U:%G" /etc/kubernetes/manifests/kube-apiserver.yaml
chmod 600 /etc/kubernetes/pki/*.key
chown root:root /etc/kubernetes/admin.conf
```

### Quick Decision Table

| Goal | Tool |
|---|---|
| Restrict syscalls | **Seccomp** |
| Restrict file paths / network / capabilities via MAC | **AppArmor** |
| Restrict Linux capabilities (setuid, net admin, etc.) | `securityContext.capabilities` |
| Prevent root / privilege escalation | `runAsNonRoot`, `allowPrivilegeEscalation: false` |
| Audit node config against CIS benchmarks | `kube-bench` |
| Detect runtime anomalies (syscalls, file access) | `Falco` |

---

## Question Wording → Tool/Command Mapping

CKS questions rarely say "use AppArmor" or "use Seccomp" directly — they describe a *goal*, and you're expected to know which tool solves it. Here's how to decode common phrasing:

### User / Group / Access related

| Question phrasing | What it actually wants |
|---|---|
| "Create a local user ... and restrict their access to the cluster" | `useradd` (or client cert/kubeconfig) + **RBAC** (Role/RoleBinding) |
| "Ensure the user can only ... in namespace X" | `Role` + `RoleBinding` scoped to that namespace, verify with `kubectl auth can-i --as=` |
| "Generate a client certificate / CSR for user" | `openssl req` + `kubectl certificate approve` |
| "Restrict SSH / login access to node" | `usermod -L`, `passwd -l`, or sshd_config changes — **not** Kubernetes objects |

### Hardening / Syscall / Profile related

| Question phrasing | What it actually wants |
|---|---|
| "Restrict the syscalls a container can make" | **Seccomp** profile (`securityContext.seccompProfile`) |
| "Prevent the container from performing certain system calls" | **Seccomp** |
| "Apply mandatory access control" / "restrict file access, network access, or capabilities of a process using a security module" | **AppArmor** |
| "Load / enforce a profile on the node" | `apparmor_parser`, `aa-enforce` |
| "Audit the syscalls without blocking them" | Seccomp profile with `type: Localhost` + **complain/audit-style JSON** (action `SCMP_ACT_LOG`, not `ERRNO`) |
| "Ensure the container runs with the default secure profile" | `seccompProfile.type: RuntimeDefault` |
| "Drop unnecessary Linux capabilities" | `securityContext.capabilities.drop: ["ALL"]` + `add:` only what's needed |
| "Prevent privilege escalation" | `allowPrivilegeEscalation: false` |
| "Ensure the container does not run as root" | `runAsNonRoot: true`, `runAsUser:` |
| "Make the root filesystem immutable / read-only" | `readOnlyRootFilesystem: true` |
| "Disable a kernel module on the node" | `/etc/modprobe.d/blacklist.conf` + `rmmod`/`lsmod` |
| "Harden kernel parameters" | `sysctl -w`, `/etc/sysctl.d/` |

### Audit / Compliance related

| Question phrasing | What it actually wants |
|---|---|
| "Audit the cluster / node against CIS benchmarks" | `kube-bench` |
| "Fix the file permissions flagged by the benchmark" | `chmod`/`chown` on the specific file named in output |
| "Detect anomalous behavior at runtime" | **Falco** |
| "Scan the image for vulnerabilities before deployment" | **Trivy** |
| "Ensure only signed images can run" | image signing / admission controller (e.g. Kyverno, cosign verification) |

### General exam-reading tips

- **"Restrict what a process/container can do at the OS/kernel level"** → think Seccomp (syscalls) or AppArmor (files/network/capabilities) — the question usually distinguishes by mentioning *syscalls* (Seccomp) vs *file paths, network, or general access control* (AppArmor).
- **"Without modifying the application"** → almost always a `securityContext` or profile-based fix, not code changes.
- **"On the node"** vs **"in the pod spec"** → tells you whether the fix belongs in a host-level file (`/etc/...`, kubelet config) or in YAML.
- Command verbs like *"create", "ensure", "configure", "restrict", "enforce", "verify"* each map to a predictable action — "verify" usually means you also need to run a check command (`kubectl describe`, `cat /proc/.../status`, `dmesg`) and not just apply the YAML.

---

## Exam Notes

- Documentation allowed in the CKS exam: `kubernetes.io/docs` and subdomains, `github.com/kubernetes` and subdomains, `kubernetes.io/blog` and subdomains, plus task-specific links provided in the Quick Reference box for each question (e.g. Falco, Trivy, AppArmor docs).
- Plain Linux `useradd`/`usermod`/`passwd` syntax is **not** documented in any allowed exam resource — memorize it.
- Always verify a security change actually took effect (`kubectl describe pod`, `cat /proc/1/status`, `dmesg`) before moving to the next task — partial credit isn't given for YAML that looks right but wasn't applied correctly.