`kubectl proxy` - Opens proxy port to API server
`kubectl proxy &` - run in the background
`kubectl proxy --port=8002 &` -- custom port to run and default port is 8001

`kubectl port-forward` - Opens port to target deployment pods
`k port-forward pod/nginx-77bc6bd484-b5jkb 8005:80 &` 
`k port-forward pod/<podname> remoteport:localport`
