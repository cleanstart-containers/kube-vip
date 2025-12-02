# Kube-VIP Container

This container package ships the custom `cleanstart/kube-vip:latest-dev` image
and a ready-to-run Kubernetes sample that demonstrates how to provide
LoadBalancer functionality in environments without a cloud load balancer
integration (for example Kind or on-prem clusters).

## Contents

- `kubernetes/deployment.yaml` – complete manifest bundle for RBAC, ConfigMap,
  DaemonSet, sample app, and service.
- `kubernetes/README.md` – detailed, step-by-step walkthrough for deploying and
  testing kube-vip on a Kind cluster.

## Prerequisites

- Running Kubernetes cluster (Kind recommended for local testing)
- `kubectl` configured for the target cluster
- Docker (needed for inspecting the image and Kind networking details)
- Network access to the cluster nodes

## Quick Start

1. **Inspect the image (optional)** – confirm the kube-vip binary is present by
   inspecting the container image with your preferred tooling.
2. **Configure the IP pool** – create the ConfigMap described in
   `kubernetes/README.md` to define the VIP CIDR range that kube-vip can assign.
3. **Deploy the sample** – apply the manifest bundle at
   `kubernetes/deployment.yaml` to install the RBAC, DaemonSet, test app, and
   LoadBalancer service.
4. **Verify** – ensure pods in namespace `kube-vip-sample` come up healthy and
   confirm that the `test-loadbalancer` service reports an external IP from your
   configured range.
5. **Exercise load balancing** – send repeated HTTP requests to the VIP (or via
   port-forwarding) and observe responses alternating between the two sample
   backend pods.

The `kubernetes/README.md` file documents the exact commands, expected output,
and validation steps for each stage of the workflow.

## Project Highlights

- **Purpose-built image** – ships a trimmed `cleanstart/kube-vip:latest-dev`
  image with the kube-vip binary installed under `/usr/bin/kube-vip`.
- **Fully scripted sample** – a single manifest bundle wires together namespace,
  RBAC, DaemonSet, ConfigMap-backed configuration, and a demo HTTP service.
- **Kind-friendly defaults** – configuration values (such as the sample VIP
  CIDR) align with the default Kind Docker network and work out of the box for
  local clusters.
- **Observability baked in** – the sample highlights how to tail kube-vip logs,
  confirm leader election status, and verify service endpoints, making it easy
  to adapt for production scenarios.

## What Gets Deployed

- A dedicated namespace (`kube-vip-sample`) with least-privilege RBAC for
  kube-vip operations.
- A ConfigMap that seeds kube-vip with a small pool of assignable Virtual IPs.
- A DaemonSet running kube-vip on every node to handle leader election and IP
  assignment.
- A simple Deployment that exposes two replica pods behind a ClusterIP service.
- A LoadBalancer service that kube-vip updates with an external VIP, allowing
  you to validate traffic routing and failover behaviour.

## Cleanup

Remove the deployed resources by deleting the manifest bundle and its supporting
ConfigMap. The namespace `kube-vip-sample` should terminate automatically; if
it does not, list the remaining resources in that namespace and clean them up
manually. Detailed instructions are included in `kubernetes/README.md`.

## Troubleshooting

- **Pods in CrashLoopBackOff** – review kube-vip logs for permission or
  interface issues, and confirm the binary is present in the container image.
- **External IP stuck on `<pending>`** – check that the `kubevip` ConfigMap
  exists in `kube-system` and that RBAC allows updating `services/status`.
- **Host cannot reach VIP directly** – reach the service via port-forwarding or
  by running curl/wget from within the cluster network.

Detailed diagnostics and commands are documented in
`kubernetes/README.md`.

