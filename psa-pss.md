# Pod security Admission & Pod Security Standards 

earlier it was pod security policy but after v1.25 it become 

PSA configure at the namespace level
Three security standard
1. privilidge -> unrestricted policy --> It allows wident possible level of permission, almost like there is no restriction
2. baseline -> minimallly restricted policy --> 
3. restricted -> heavily restricted policy --> It resticted level permission, follow like a pod hardening best practices.

The PSA Mode -> on violation
1. enforce -> reject Pod
2. audit -> record in audit logs
3. warn -> trigger user-facing warning

How to use this in namespace level using the labels 
`k label ns payroll pod-security.kubernetes.io/enforce=restricted` 
here we use the <mode>=<standard> in any combination ex. pod-security.kubernetes.io/audit=restricted 
                                                         pod-security.kubernetes.io/audit=baseline
                                                         pod-security.kubernetes.io/warn=priviledge

