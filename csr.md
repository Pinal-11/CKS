# Challenge: Validate User Access and Fix the Problems

In this cluster, there are multiple users and contexts. A user named `user-dev` within the context `user-dev-ctx` cannot access pods in the `default` namespace. There should be anything wrong regarding the access.

Check and resolve the access issues so that `user-dev` is able to **get, watch, and list** pods in the namespace `default`.

---

## Solution

### 1. Switch Context and Test View Pods in Namespace `default`

```bash
kubectl config get-contexts               # See the available contexts
kubectl config use-context user-dev-ctx   # Change context
kubectl get pod
```

You will get response:
```
error: You must be logged in to the server (Unauthorized)
```

### 2. Check the User Context

```bash
kubectl config view --raw
```

As you can see the result below, `user-dev-ctx` belongs to `user-dev` and it is a certificate-based user.

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.30.1.2:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
- context:
    cluster: kubernetes
    user: user-dev
  name: user-dev-context
current-context: user-dev-context
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
- name: user-dev
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
```

### 3. Check the Certificate Details

It may have expired, since the previous error says `Unauthorized`:

```bash
kubectl config view --raw -o jsonpath='{.users[1].user.client-certificate-data}' | base64 -d | openssl x509 -noout -text
```

Look at the `Validity` section — the user's certificate is expired!

```
Validity
    Not Before: Sep  1 00:00:00 2024 GMT
    Not After : Sep  1 00:00:00 2025 GMT
```

**Hint:** After step 1, when you get an `Unauthorized` response, switch back to `kubernetes-admin@kubernetes` and check the `kube-apiserver` logs for more verbose output.

```bash
kubectl config use-context kubernetes-admin@kubernetes
kubectl logs -n kube-system kube-apiserver-controlplane
```

```
E0924 11:25:36.344326       1 authentication.go:75] "Unable to authenticate the request" err="[x509: certificate has expired or is not yet valid: current time 2025-09-24T11:25:36Z is after 2025-09-01T00:00:00Z, verifying certificate SN=640911260284410447831826718592779922701119607375, SKID=, AKID= failed: x509: certificate has expired or is not yet valid: current time 2025-09-24T11:25:36Z is after 2025-09-01T00:00:00Z]"
```

The error confirms it: the user certificate has expired.

### 4. Rotate the Certificate for `user-dev`

```bash
# Create user certificate pair for rotating the expired cert
openssl genrsa -out user-dev.key 4096
openssl req -new -key user-dev.key -out user-dev.csr -subj "/CN=user-dev"

# As kubernetes-admin@kubernetes, create a Kubernetes CertificateSigningRequest
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: user-dev-csr
spec:
  request: <base64-encoded-csr-file>  # $(cat user-dev.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF

# Approve the CSR
kubectl certificate approve user-dev-csr

# Export the issued certificate
kubectl get csr user-dev-csr -o jsonpath='{.status.certificate}'| base64 -d > user-dev.crt

# Rotate the user certificate
kubectl config set-credentials user-dev --client-key=user-dev.key --client-certificate=user-dev.crt --embed-certs=true
```

### 5. Verify Step 1 Again

```bash
kubectl config use-context user-dev-ctx
kubectl get pod
```

```
Error from server (Forbidden): pods is forbidden: User "user-dev" cannot list resource "pods" in API group "" in the namespace "default"
```

A different error compared to step 1 — the expired cert issue is solved. Now the forbidden access needs to be fixed. This is a Kubernetes **RBAC** topic.

### 6. Check Role and RoleBinding

```bash
# Always use kubernetes-admin@kubernetes for administrating purpose
kubectl config use-context kubernetes-admin@kubernetes

kubectl get role,rolebinding
```

```
No resources found in default namespace.
```

We need to create a new role and bind it to `user-dev` in namespace `default`.

```bash
kubectl create role pod-viewer -n default --verb get,watch,list --resource pod
kubectl create rolebinding pod-viewer-bind -n default --role pod-viewer --user user-dev
```

```bash
kubectl get role,rolebinding # Check it
```

```
NAME                                        CREATED AT
role.rbac.authorization.k8s.io/pod-viewer   2025-09-24T12:10:44Z

NAME                                                       ROLE              AGE
rolebinding.rbac.authorization.k8s.io/pod-viewer-binding   Role/pod-viewer   4s
```

### 7. Validate the Fixes

```bash
kubectl config use-context user-dev-context
```
```
Switched to context "user-dev-context".
```

```bash
kubectl get pod
```
```
No resources found in default namespace.
```

All done — both problems are fixed. This is the expected return since there are no pods in the `default` namespace, but `user-dev` now has access.