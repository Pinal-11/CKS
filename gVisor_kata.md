# gVisor Containers

Containerization has one core problem is that they all interact pretty much directly with the same os, and especially the same kernel.
So we want to improve the isolation between the container-to-container and container-to-os/kernel.

gVisor is a tool form google that allows an additional leyar of isolation between the container and the kernel.

gVisor component -> Sentry & Gofer

```
                                gVisor
                           |-------------|
Container <-> syscall() -> Sentry <-> gofer -> linux kernel -> hardware 
                              |  
                              V
                        Limited Syscall   
```

**The Main takeway:-** Its not just one single gVisor kernel that serves all the containers. Each containerezed env has its own dedicated gVisor kernel acting as the middleman between the application and the Linux kernel.

This means that each container would now be isolated in its own virtualized sandbox which drastically reduces the attack surface.

One more disadvantage is -> since the system calls are received and processed via a middleman, this means that there are more instructions the cpu has to go through. This makes the application slighter slower as compared to traditional containers.

**gVisor runtime -->** `runsc`

---

# Kata Containers

Kata inserts each container into its own seperate virtual machine. And each container will have its own dedicated kernel running inside.

Just like gVisor, this gets rid of problems caused when all containers app communicate directly to the same OS kernel. Now, each container has its own little kernel to bother. So if they crash it or abuse it in any way it doesn't bring the whole system down. Only that specific container will experience problems instead of all running containers on the system.

**Kata container runtime -->** `kata-runtime`

---

# Container Runtime

Both Kata and gVisor are compatible with the OCI (Open Container Initiative).

```bash
docker run -d nginx
docker run --runtime kata -d nginx
docker run --runtime runsc -d nginx
```

![docker container runtime](<SS/Screenshot 2026-09-11 at 12.34.32 PM.png>)
![runc](<SS/Screenshot 2026-09-11 at 12.35.01 PM.png>)

---

## gVisor.yaml

```yaml
apiVersion: node.k8s.io/v1beta1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
```

## pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  runtimeClassName: gvisor
  containers:
  - name: nginx
    image: nginx
```