# Service Account

It is generally use for the application bot to run inside the cluster with the proper permission.
Any pod is created the by default its create the default service account and it is projected as a volume inside the pod. and the location would be `/var/run/secrets/kubernetes.io/serviceaccount` inside the pod.

### Commands
`kubectl create sa <sa-name> -n <namespace>`
`k get sa <name>`
`k describe sa <name>`

---------------------

* Service Account are used by other applications or service to interact with k8s 
* Tokens are created for service account to prove identity
* To create SA `k create sa <sa-name>`
* To use the SA from an exteranl application (CI/CD, monitoring tools) `k create token <SA-name> --duration 2h`

* every Namespace has a their own default service account 
* The default service account is automatically attach to Pod on creations.
* To attach a service account to Pod use `spec.serviceAccountName` field
* When service account is attached to a Pod, K8s:
    * Automatically creates a token and mounts as a Projected Volume.
    * Automatically rotate Tokens
    * Automatically expire the token when the Pod is deleted.

* For not Automount use the `spec.automountServiceAccountToken: false` field in the Pod and you may also define the same thing at the ServiceAccount Level as well.