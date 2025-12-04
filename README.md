# Kube-VIP Container - CleanStart

This container package ships the custom `cleanstart/kube-vip:latest-dev` image and a ready-to-run Kubernetes sample that demonstrates how to provide LoadBalancer functionality in environments without a cloud load balancer integration (for example Kind or on-prem clusters). The CleanStart Kube-VIP image provides a production-ready, security-hardened container optimized for enterprise environments. Built on a minimal base OS with comprehensive security hardening, this image delivers reliable application execution with advanced security features.

**📌 CleanStart Foundation:** Security-hardened, minimal base OS designed for enterprise containerized environments.

**Image Path:** `cleanstart/kube-vip`

**Registry:** CleanStart Registry

---

## Overview

Kube-VIP provides LoadBalancer functionality for Kubernetes clusters that don't have native cloud load balancer integration. It manages Virtual IPs (VIPs) for services and performs leader election to ensure high availability. This CleanStart Kube-VIP container is part of the CleanStart application suite, featuring enterprise-grade security hardening, automated vulnerability management, and compliance with industry standards.

---

## About CleanStart

CleanStart is a comprehensive container registry providing security-hardened, enterprise-ready container images. Our images are designed with security-first principles, featuring minimal attack surfaces, regular security updates, and compliance with industry standards.

### About CleanStart Images

CleanStart images are built on secure, minimal base operating systems and optimized for production environments. Each image undergoes rigorous security testing, vulnerability scanning, and compliance validation to ensure enterprise-grade security and reliability.

---

## Contents

- `kubernetes/deployment.yaml` – complete manifest bundle for RBAC, ConfigMap, DaemonSet, sample app, and service
- `kubernetes/README.md` – detailed, step-by-step walkthrough for deploying and testing kube-vip on a Kind cluster

---

## Key Features

- **Security-First Design**: Built with minimal attack surfaces and security hardening
- **Enterprise Compliance**: Meets industry standards including FIPS, STIG, and CIS benchmarks
- **Regular Updates**: Automated security patches and vulnerability management
- **Multi-Architecture Support**: Available for AMD64 and ARM64 architectures
- **Production Ready**: Optimized for enterprise deployment and scaling
- **Comprehensive Documentation**: Detailed guides and best practices for each image
- **Purpose-built image**: Ships a trimmed `cleanstart/kube-vip:latest-dev` image with the kube-vip binary installed under `/usr/bin/kube-vip`
- **Fully scripted sample**: A single manifest bundle wires together namespace, RBAC, DaemonSet, ConfigMap-backed configuration, and a demo HTTP service
- **Kind-friendly defaults**: Configuration values (such as the sample VIP CIDR) align with the default Kind Docker network and work out of the box for local clusters
- **Observability baked in**: The sample highlights how to tail kube-vip logs, confirm leader election status, and verify service endpoints

---

## Prerequisites

Before deploying, ensure you have:

- Running Kubernetes cluster (Kind recommended for local testing)
- `kubectl` configured for the target cluster
- Docker (needed for inspecting the image and Kind networking details)
- Network access to the cluster nodes

---

## Quick Start

### Pull Commands
```bash
docker pull cleanstart/kube-vip:latest
docker pull cleanstart/kube-vip:latest-dev
```

### Run Commands

Basic test:
```bash
docker run -it --name kube-vip-test cleanstart/kube-vip:latest-dev
```

Production deployment:
```bash
docker run -d --name kube-vip-prod \
  --read-only \
  --security-opt=no-new-privileges \
  --user 1000:1000 \
  cleanstart/kube-vip:latest
```

### Deployment Steps

1. **Inspect the image (optional)** – confirm the kube-vip binary is present by inspecting the container image with your preferred tooling

2. **Configure the IP pool** – create the ConfigMap described in `kubernetes/README.md` to define the VIP CIDR range that kube-vip can assign

3. **Deploy the sample** – apply the manifest bundle at `kubernetes/deployment.yaml` to install the RBAC, DaemonSet, test app, and LoadBalancer service

4. **Verify** – ensure pods in namespace `kube-vip-sample` come up healthy and confirm that the `test-loadbalancer` service reports an external IP from your configured range

5. **Exercise load balancing** – send repeated HTTP requests to the VIP (or via port-forwarding) and observe responses alternating between the two sample backend pods

The `kubernetes/README.md` file documents the exact commands, expected output, and validation steps for each stage of the workflow.

---

## What Gets Deployed

- A dedicated namespace (`kube-vip-sample`) with least-privilege RBAC for kube-vip operations
- A ConfigMap that seeds kube-vip with a small pool of assignable Virtual IPs
- A DaemonSet running kube-vip on every node to handle leader election and IP assignment
- A simple Deployment that exposes two replica pods behind a ClusterIP service
- A LoadBalancer service that kube-vip updates with an external VIP, allowing you to validate traffic routing and failover behaviour

---

## Architecture Support

CleanStart images support multiple architectures to ensure compatibility across different deployment environments:

- **AMD64**: Intel and AMD x86-64 processors
- **ARM64**: ARM-based processors including Apple Silicon and ARM servers

### Architecture-based Pull Commands
```bash
docker pull --platform linux/amd64 cleanstart/kube-vip:latest
docker pull --platform linux/arm64 cleanstart/kube-vip:latest
```

---

## Troubleshooting

Common issues and solutions:

- **Pods in CrashLoopBackOff** – review kube-vip logs for permission or interface issues, and confirm the binary is present in the container image
- **External IP stuck on `<pending>`** – check that the `kubevip` ConfigMap exists in `kube-system` and that RBAC allows updating `services/status`
- **Host cannot reach VIP directly** – reach the service via port-forwarding or by running curl/wget from within the cluster network

Detailed diagnostics and commands are documented in `kubernetes/README.md`.

---

## Cleanup

Remove the deployed resources by deleting the manifest bundle and its supporting ConfigMap. The namespace `kube-vip-sample` should terminate automatically; if it does not, list the remaining resources in that namespace and clean them up manually. Detailed instructions are included in `kubernetes/README.md`.

---

## Resources

- **Official Documentation:** https://kube-vip.io/
- **Kube-VIP GitHub Repository:** https://github.com/kube-vip/kube-vip
- **Provenance / SBOM / Signature:** https://images.cleanstart.com/images/kube-vip
- **Docker Hub:** https://hub.docker.com/r/cleanstart/kube-vip
- **CleanStart All Images:** https://images.cleanstart.com
- **CleanStart Community Images:** https://hub.docker.com/u/cleanstart

---

## Vulnerability Disclaimer

CleanStart offers Docker images that include third-party open-source libraries and packages maintained by independent contributors. While CleanStart maintains these images and applies industry-standard security practices, it cannot guarantee the security or integrity of upstream components beyond its control.

Users acknowledge and agree that open-source software may contain undiscovered vulnerabilities or introduce new risks through updates. CleanStart shall not be liable for security issues originating from third-party libraries, including but not limited to zero-day exploits, supply chain attacks, or contributor-introduced risks.

**Security remains a shared responsibility:** CleanStart provides updated images and guidance where possible, while users are responsible for evaluating deployments and implementing appropriate controls.
