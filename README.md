# K8s-labs

# Kubernetes Live Labs

![Kubernetes](https://img.shields.io/badge/Kubernetes-Live%20Labs-326CE5?logo=kubernetes&logoColor=white)
![Platform](https://img.shields.io/badge/Platform-Killercoda-F97316)

A hands-on Kubernetes learning repository containing practical labs completed in live Killercoda environments.

The purpose of this repository is to demonstrate how Kubernetes resources are created, configured, tested and troubleshooted—not simply to provide a collection of commands and YAML files.

## Lab approach

Each lab follows the same practical workflow:

> **Objective → Create → Apply → Verify → Test → Troubleshoot → Clean up → Reflect**

Every module will include:

- The objective and Kubernetes concepts covered
- YAML manifests
- Commands used in Killercoda
- Verification and application testing
- Troubleshooting notes
- Cleanup commands
- A summary of what I learned

## Lab environment

| Component | Details |
|---|---|
| Lab platform | Killercoda |
| Cluster | Kubernetes lab cluster |
| Command-line tool | kubectl |
| Container runtime | containerd |
| Configuration | Kubernetes YAML manifests |

> Killercoda environments are temporary. This repository preserves the manifests, explanations and selected evidence produced during each live lab.

## Learning path

| # | Module | Topics |
|---:|---|---|
| 01 | Pods, Deployments and Controllers | Pods, Deployments, ReplicaSets and controllers |
| 02 | Services and Load Balancing | Services, ClusterIP, NodePort and LoadBalancer |
| 03 | ConfigMaps and Secrets | ConfigMaps, Secrets and environment variables |
| 04 | Persistent Kubernetes Storage | Volumes, PersistentVolumes, PersistentVolumeClaims and StorageClasses |
| 05 | Networking, DNS and NetworkPolicies | Pod communication, DNS, CNI and NetworkPolicies |
| 06 | Ingress and External Access | Ingress controllers, TLS and routing |
| 07 | RBAC, ServiceAccounts and Pod Security | Access control and workload security |
| 08 | Kyverno and Policy Enforcement | Validation, mutation and policy enforcement |
| 09 | Scheduling and Node Operations | Scheduling, affinity, taints, tolerations and node maintenance |
| 10 | Logs, Events and Debugging | Metrics, logs, events and workload failures |
| 11 | Service Mesh with Istio | Istio, Kiali, traffic management and mTLS |
| 12 | Dashboard and Cluster Administration | Kubernetes Dashboard, kubectl and cluster resources |

## Repository structure

```text
kubernetes-live-labs/
├── README.md
├── 01-workloads/
├── 02-services/
├── 03-configuration/
├── 04-storage/
├── 05-networking/
├── 06-ingress/
├── 07-security/
├── 08-policy/
├── 09-scheduling/
├── 10-observability/
├── 11-service-mesh/
└── 12-administration/
```

Each module will contain its own README, Kubernetes manifests and selected evidence from the live environment.

## Evidence standard

A resource returning `created` is not enough to prove that it works. Each lab will include verification of the resource state and a practical behaviour test.

Evidence may include:

- Resource manifests
- Relevant terminal commands and output
- Pod, Deployment and Service information
- Application responses
- Kubernetes events and logs
- Scaling, recovery or failure behaviour
- Troubleshooting and resolutions

Sensitive information such as credentials, tokens, certificates and kubeconfig contents will not be committed.

## Topics covered

The labs progress from Kubernetes fundamentals to more advanced cluster operations:

- Workload creation and management
- Internal and external service exposure
- Application configuration
- Persistent storage
- Kubernetes networking and DNS
- Ingress routing
- Identity, permissions and workload security
- Policy enforcement
- Workload scheduling
- Observability and troubleshooting
- Service mesh concepts
- Cluster administration

## Disclaimer

This is a personal learning and portfolio repository. The examples are designed for temporary Kubernetes lab environments and should be reviewed before being used in production.
