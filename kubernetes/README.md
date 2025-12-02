# Kube-VIP Sample Project - Complete Guide

## Overview
This project demonstrates how to deploy kube-vip in a Kubernetes cluster to provide LoadBalancer functionality using the custom `cleanstart/kube-vip:latest-dev` image.

**What is Kube-VIP?**
Kube-VIP provides Virtual IP (VIP) addresses for Kubernetes Services of type LoadBalancer using ARP or BGP, perfect for on-premises or Kind clusters without cloud provider integration.

---

## Prerequisites

- Kubernetes cluster (Kind recommended)
- kubectl installed and configured
- Docker installed
- Network access to the cluster

---

## Step-by-Step Deployment

### Step 1: Verify Your Cluster
Check your cluster is running:
```bash
kubectl cluster-info
```

Check your nodes:
```bash
kubectl get nodes
```

Expected output:
```
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   35d   v1.30.0
```

---

### Step 2: Identify Network Configuration

For Kind clusters, find the Docker network subnet:
```bash
docker network inspect kind | grep Subnet
```

Expected output:
```
"Subnet": "172.18.0.0/16"
```

Check node IP:
```bash
kubectl get nodes -o wide
```

Note the INTERNAL-IP (e.g., `172.18.0.4`)

---

### Step 3: Verify Image Configuration

Check that the kube-vip binary exists in the correct location:
```bash
docker run --rm --entrypoint /bin/sh cleanstart/kube-vip:latest-dev -c "ls -la /usr/bin/kube-vip"
```

Expected output:
```
-rwxr-xr-x    1 root     root      [size] [date] /usr/bin/kube-vip
```

---

### Step 4: Create ConfigMap for IP Pool

Create a ConfigMap in kube-system namespace to define the IP range:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubevip
  namespace: kube-system
data:
  cidr-global: "172.18.0.200/29"
EOF
```

Verify creation:
```bash
kubectl get configmap -n kube-system kubevip
```

---

### Step 5: Deploy All Resources

Apply the deployment file:
```bash
kubectl apply -f deployment.yaml
```

Expected output:
```
namespace/kube-vip-sample created
serviceaccount/kube-vip created
clusterrole.rbac.authorization.k8s.io/kube-vip-sample-role created
clusterrolebinding.rbac.authorization.k8s.io/kube-vip-sample-binding created
configmap/kube-vip-config created
daemonset.apps/kube-vip created
service/test-loadbalancer created
deployment.apps/test-app created
```

---

### Step 6: Verify Pods are Running

Check all pods:
```bash
kubectl get pods -n kube-vip-sample
```

Expected output:
```
NAME                        READY   STATUS    RESTARTS   AGE
kube-vip-xxxxx              1/1     Running   0          30s
test-app-xxxxxxx-xxxxx      1/1     Running   0          30s
test-app-xxxxxxx-xxxxx      1/1     Running   0          30s
```

Wait until all pods show `Running` status.

---

### Step 7: Check Kube-VIP Logs

View kube-vip startup logs:
```bash
kubectl logs -n kube-vip-sample -l app=kube-vip
```

Look for success messages:
```
INFO successfully acquired lease kube-vip-sample/plndr-svcs-lock
INFO (svcs) adding VIP ip=172.18.0.200 interface=eth0
INFO layer 2 broadcaster starting
```

If you see errors about permissions, the RBAC rules need adjustment (already fixed in deployment.yaml).

---

### Step 8: Verify LoadBalancer External IP

Check the service:
```bash
kubectl get svc -n kube-vip-sample test-loadbalancer
```

Expected output:
```
NAME                TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
test-loadbalancer   LoadBalancer   10.96.76.208   172.18.0.200   80:31123/TCP   2m
```

The EXTERNAL-IP should show `172.18.0.200` (not `<pending>`).

---

### Step 9: Test from Within the Cluster

The LoadBalancer IP works within the Kubernetes cluster network:
```bash
kubectl run -it --rm debug --image=alpine --restart=Never -- sh
```

Inside the pod, test the service:
```bash
wget -O- http://172.18.0.200
exit
```

Expected output:
```
<h1>Hello from Kube-VIP Test App</h1><p>Pod: test-app-xxxxxxx-xxxxx</p>
```

---

### Step 10: Test Load Balancing

Run multiple requests to see different pods responding:
```bash
kubectl run -it --rm debug --image=alpine --restart=Never -- sh -c "for i in 1 2 3 4 5; do wget -O- http://172.18.0.200 2>/dev/null; echo ''; done"
```

You should see different pod names in the responses, proving load balancing works!

---

### Step 11: Access from Host Machine

Since the LoadBalancer IP is in the Kind Docker network, access from your host requires port-forwarding:

**Terminal 1:** Start port-forward
```bash
kubectl port-forward -n kube-vip-sample svc/test-loadbalancer 8080:80
```

**Terminal 2:** Test the service
```bash
curl http://localhost:8080
```

Expected output:
```
<h1>Hello from Kube-VIP Test App</h1><p>Pod: test-app-xxxxxxx-xxxxx</p>
```

Test load balancing:
```bash
for i in {1..10}; do curl http://localhost:8080; echo ""; done
```

---

### Step 12: Verify Service Details

Get detailed service information:
```bash
kubectl describe svc -n kube-vip-sample test-loadbalancer
```

Check endpoints:
```bash
kubectl get endpoints -n kube-vip-sample test-loadbalancer
```

Expected:
```
NAME                ENDPOINTS                           AGE
test-loadbalancer   10.244.0.20:8080,10.244.0.21:8080   5m
```

---

## Cleanup

### Remove All Resources
```bash
kubectl delete -f deployment.yaml
```

### Remove IP Pool ConfigMap
```bash
kubectl delete configmap -n kube-system kubevip
```

### Verify Deletion
```bash
kubectl get namespace kube-vip-sample
```

Expected:
```
Error from server (NotFound): namespaces "kube-vip-sample" not found
```

---

## Troubleshooting

### Problem: Kube-VIP Pod CrashLoopBackOff

**Check logs:**
```bash
kubectl logs -n kube-vip-sample -l app=kube-vip --tail=50
```

**Common causes:**
1. **Binary not found** - Verify `/usr/bin/kube-vip` exists in image
2. **Interface error** - Check `eth0` exists: `docker exec kind-control-plane ip addr show`
3. **Permissions error** - Verify RBAC includes `services/status` update permission

---

### Problem: External IP Stuck on `<pending>`

**Check kube-vip logs:**
```bash
kubectl logs -n kube-vip-sample -l app=kube-vip
```

**If you see permission errors:**
```
ERROR updating Service ... cannot update resource "services/status"
```

The RBAC rules need `services/status` update permission (already included in deployment.yaml).

**Check ConfigMap exists:**
```bash
kubectl get configmap -n kube-system kubevip -o yaml
```

**Restart kube-vip:**
```bash
kubectl delete pod -n kube-vip-sample -l app=kube-vip
```

---

### Problem: Cannot Access LoadBalancer from Host

**This is expected behavior** - The LoadBalancer IP (`172.18.0.200`) is in the Kind Docker network and not directly accessible from your host machine.

**Solutions:**

1. **Use port-forward (Recommended):**
```bash
kubectl port-forward -n kube-vip-sample svc/test-loadbalancer 8080:80
curl http://localhost:8080
```

2. **Test from within cluster:**
```bash
kubectl run -it --rm debug --image=alpine --restart=Never -- wget -O- http://172.18.0.200
```

3. **Use Docker network (Advanced):**
```bash
docker run --rm --network kind alpine wget -O- http://172.18.0.200
```

---

### Problem: Pods Not Starting

**Describe the pod:**
```bash
kubectl describe pod -n kube-vip-sample -l app=kube-vip
```

**Check events:**
```bash
kubectl get events -n kube-vip-sample --sort-by='.lastTimestamp'
```

**Common issues:**
- Image pull errors - Verify image exists: `docker images | grep kube-vip`
- Resource constraints - Check node resources: `kubectl top nodes`

---

## How It Works

1. **DaemonSet Deployment**
   - Kube-vip runs on each node (in this case, only control-plane)

2. **Leader Election**
   - One kube-vip instance becomes leader using Kubernetes leases
   - Leader is responsible for managing VIPs

3. **Service Watch**
   - Kube-vip watches for LoadBalancer services
   - When detected, assigns IP from configured range

4. **IP Assignment**
   - Checks service annotation `kube-vip.io/loadbalancerIPs`
   - If not specified, allocates from global CIDR pool

5. **ARP Advertisement**
   - Leader binds VIP to specified interface (`eth0`)
   - Responds to ARP requests for the VIP
   - Updates service status with External IP

6. **Traffic Flow**
   - External traffic to VIP reaches the leader node
   - Kubernetes Service routes traffic to backend pods
   - Load balancing happens at Service level

---

**Project Status:** ✅ Fully Functional

The kube-vip deployment successfully provides LoadBalancer functionality within your Kubernetes cluster!