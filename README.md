# K3S Learning

## Prerequisites

### K3d
K3d is used for quick setup of K3s clusters and nodes.
```bash
wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

### K9s
K9s is a terminal-based UI for managing Kubernetes clusters.
```bash
wget https://github.com/derailed/k9s/releases/latest/download/k9s_linux_amd64.deb && apt install ./k9s_linux_amd64.deb && rm k9s_linux_amd64.deb
```

## Creating a Cluster

Create a cluster with 2 agent nodes:
```bash
k3d cluster create mycluster --agents 2
```

Verify cluster creation:
```bash
$ k3d cluster list
NAME        SERVERS   AGENTS   LOADBALANCER
mycluster   1/1       2/2      true
```

Check nodes status:
```bash
$ kubectl get nodes -o wide
NAME                     STATUS   ROLES                  AGE    VERSION        INTERNAL-IP   EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION     CONTAINER-RUNTIME
k3d-mycluster-agent-0    Ready    <none>                 112s   v1.31.5+k3s1   172.19.0.3    <none>        K3s v1.31.5+k3s1   6.8.0-63-generic   containerd://1.7.23-k3s2
k3d-mycluster-agent-1    Ready    <none>                 108s   v1.31.5+k3s1   172.19.0.4    <none>        K3s v1.31.5+k3s1   6.8.0-63-generic   containerd://1.7.23-k3s2
k3d-mycluster-server-0   Ready    control-plane,master   2m     v1.31.5+k3s1   172.19.0.2    <none>        K3s v1.31.5+k3s1   6.8.0-63-generic   containerd://1.7.23-k3s2
```

## Resource Limiting

### Limitation: Docker vs Kubernetes Resource Awareness

**Important**: Docker container resource limits and Kubernetes node capacity are separate concepts:
- Docker limits: Enforced at runtime by the container runtime
- Kubernetes capacity: Read from the host system, not from Docker limits

This means Kubernetes will schedule pods based on host resources (32GB RAM, 16 CPU) even if Docker containers are limited to smaller amounts.

### Prevent Master Node Scheduling

To ensure pods are only scheduled on worker nodes (not the master):
```bash
kubectl taint nodes k3d-mycluster-server-0 node-role.kubernetes.io/control-plane=true:NoSchedule
```
**if not master node can be serve some pod if the resource available**

### K3s with Kubelet System Reserved (Proper Solution)

Use kubelet's `--system-reserved` flag to make Kubernetes aware of reduced node capacity:

```bash
# Create cluster with proper resource limits
k3d cluster delete mycluster

k3d cluster create mycluster \
  --agents 2 \
  --k3s-arg "--kubelet-arg=system-reserved=cpu=15,memory=30Gi@agent:0" \
  --k3s-arg "--kubelet-arg=system-reserved=cpu=14,memory=28Gi@agent:1"

# This tells kubelet:
# agent-0: Reserve 15 CPU & 30Gi → Leaves 1 CPU & 2Gi for pods
# agent-1: Reserve 14 CPU & 28Gi → Leaves 2 CPU & 4Gi for pods
```

### Node Resource Configuration
With Method 2 (system-reserved), Kubernetes will see:
- **k3d-mycluster-agent-0**: 2GB RAM, 1 vCPU allocatable
- **k3d-mycluster-agent-1**: 4GB RAM, 2 vCPU allocatable


## Testing Resource Constraints with Pods

### Pod Configurations

Three nginx pods with different resource requirements to test node scheduling:

1. **nginx-200mb-1cpu** (`nginx-200mb-1cpu.yaml`): 200MB RAM, 1 vCPU
2. **nginx-4gb-100mcpu** (`nginx-3gb-100mcpu.yaml`): 4GB RAM, 100m CPU
3. **nginx-2gb-2cpu** (`nginx-2gb-2cpu.yaml`): 2GB RAM, 2 vCPU

**so which node do you think the pod will deploy on?**

let's start: `k9s` -> :`pods` it will show all the pods informations, we will keep the terminal open to view pod in realtime

```bash
 Context: k3d-mycluster [RW]                       <0> all       <a>       Attach       <ctrl-k>  Kill            <o> Show Node         ____  __ ________        
 Cluster: k3d-mycluster                            <1> default   <ctrl-d>  Delete       <l>       Logs            <f> Show PortForward |    |/  /   __   \______ 
 User:    admin@k3d-mycluster                                    <d>       Describe     <p>       Logs Previous   <t> Transfer         |       /\____    /  ___/ 
 K9s Rev: v0.50.9                                                <e>       Edit         <shift-f> Port-Forward    <y> YAML             |    \   \  /    /\___  \ 
 K8s Rev: v1.31.5+k3s1                                           <?>       Help         <z>       Sanitize                             |____|\__ \/____//____  / 
 CPU:     1%                                                     <shift-j> Jump Owner   <s>       Shell                                         \/           \/  
 MEM:     2%                                                                                                                                                     
┌──────────────────────────────────────────────────────────────────────── pods(all)[12] ────────────────────────────────────────────────────────────────────────┐
│ NAMESPACE↑   NAME                                     PF READY STATUS     RESTARTS CPU %CPU/R %CPU/L MEM %MEM/R %MEM/L IP         NODE                    AGE │
```


now open another terminal start deploy `nginx-3gb-100mcpu.yaml`
```bash
kubectl apply -f nginx-3gb-100mcpu.yaml
```

now you will see the pod is deployed on agent-1 (look at k9s terminal)
```bash
│ NAMESPACE↑   NAME                                     PF READY STATUS     RESTARTS CPU %CPU/R %CPU/L MEM %MEM/R %MEM/L IP         NODE                    AGE │
│ default      nginx-3gb-100mcpu                        ●  1/1   Running           0   0      0      0  13      0      0 10.42.1.6  k3d-mycluster-agent-1   4m5 │
│ kube-system  coredns-ccb96694c-pjtkp                  ●  1/1   Running           0   5      5    n/a  15     22      9 10.42.0.3  k3d-mycluster-server-0  125 │
```

so agaent-1 is almost full

ok continue deploy `nginx-2gb-2cpu.yaml`

```bash
kubectl apply -f nginx-2gb-2cpu.yaml
```
```bash
│ NAMESPACE↑   NAME                                     PF READY STATUS     RESTARTS CPU %CPU/R %CPU/L MEM %MEM/R %MEM/L IP         NODE                    AGE │
│ default      nginx-2gb-2cpu                           ●  0/1   Pending           0   0      0      0   0      0      0 n/a        n/a                     9s  │
│ default      nginx-3gb-100mcpu                        ●  1/1   Running           0   0      0      0  13      0      0 10.42.1.6  k3d-mycluster-agent-1   6m4 │
│ kube-system  coredns-ccb96694c-pjtkp                  ●  1/1   Running           0   3      3    n/a  15     22      9 10.42.0.3  k3d-mycluster-server-0  128 │
```

you can see pending status, because the agent-1 is full, agent-0: only have `1 CPU & 2Gi Ram` not enough for `nginx-2gb-2cpu`

and the last we test the  pod that suitable for agent-0

```bash
kubectl apply -f nginx-200mb-1cpu.yaml
```
```bash
│ NAMESPACE↑   NAME                                     PF READY STATUS     RESTARTS CPU %CPU/R %CPU/L MEM %MEM/R %MEM/L IP         NODE                    AGE │
│ default      nginx-2gb-2cpu                           ●  0/1   Pending           0   0      0      0   0      0      0 n/a        n/a                     3m3 │
│ default      nginx-3gb-100mcpu                        ●  1/1   Running           0   0      0      0  13      0      0 10.42.1.6  k3d-mycluster-agent-1   10m │
│ default      nginx-200mb-1cpu                         ●  1/1   Running           0   0      0      0   0      0      0 10.42.2.4  k3d-mycluster-agent-0   17s │
│ kube-system  coredns-ccb96694c-pjtkp                  ●  1/1   Running           0   4      4    n/a  15     22      9 10.42.0.3  k3d-mycluster-server-0  131 │
│ kube-system  helm-install-traefik-crd-kkdrh           ●  0/1   Completed         0   0    n/a    n/a   0    n/a    n/a 10.42.0.2  k3d-mycluster-server-0  131 │
│ kube-system  helm-install-traefik-mm29s               ●  0/1   Completed         1   0    n/a    n/a   0    n/a    n/a 10.42.0.5  k3d-mycluster-server-0  131 |
```