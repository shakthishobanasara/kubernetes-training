# Kubernetes CKA Training 2025‑09

This repository contains all materials for a 10‑hour Kubernetes Administrator (CKA) training delivered in September 2025. It includes slides, detailed notes, hands‑on labs, and a practice test to prepare students for the CKA exam.

## Repository Structure

```
kubernetes-cka-training-2025-09/
│
├── slides/                  # PowerPoint deck used for the lecture
│   └── kubernetes-training.pptx
├── notes/                   # Learner and trainer notes (PDF)
│   └── kubernetes-training.pdf
├── labs/                    # Hands‑on exercises
│   ├── 00-getting-started/  # Environment setup instructions
│   ├── pods/                # Pod basics
│   ├── workloads/           # Deployments, StatefulSets, DaemonSets
│   ├── networking/          # Services and Ingress
│   ├── storage/             # PV, PVC, ConfigMaps, Secrets
│   ├── security/            # RBAC and Pod Security
│   ├── observability/       # Metrics, logging, HPA, Prometheus
│   └── advanced/            # VPA, autoscaling nodes, operators
├── practice-test/           # Practice exam with answers
│   └── practice-test.pdf
├── README.md                # This overview file
└── README.pdf               # PDF version of this file
```

## 10‑Hour Schedule

The training is divided into ten one‑hour modules:

| Hour | Module | Topics |
|----:|--------|--------|
| 1 | Fundamentals & Architecture | Motivation, cluster components, nodes |
| 2 | Pods & Deployments | Pods, ReplicaSets, Deployments |
| 3 | Workloads & Controllers | StatefulSets, DaemonSets, Jobs, CronJobs |
| 4 | Networking & Services | Service types, Ingress |
| 5 | Storage & Configuration | Volumes, PV/PVC, ConfigMaps, Secrets |
| 6 | Resource Management & Health | Requests/limits, QoS, probes |
| 7 | Scheduling & Placement | nodeSelector, affinity, taints & tolerations |
| 8 | Autoscaling & Performance | HPA, VPA, cluster autoscaler, metrics |
| 9 | Security & Access Control | RBAC, service accounts, Pod security |
| 10 | Extensions & Observability | Helm, Operators, Prometheus, troubleshooting |

## Instructor Guide

- **Slides**: Use the `kubernetes-training.pptx` deck for presentation. Speaker notes are embedded where appropriate.
- **Notes**: `notes/kubernetes-training.pdf` contains learner‑friendly explanations and trainer cues. Review this before delivering each module.
- **Labs**: Each lab directory includes step‑by‑step instructions and sample YAML manifests. Encourage participants to type commands themselves rather than copy‑paste.
- **Practice Test**: The practice test PDF contains 50 questions with detailed explanations. Use it as a knowledge check at the end or assign it as homework.
- **Setup**: Ensure every participant has a working Kubernetes cluster (Minikube or kind) before starting the hands‑on labs. Allocate time to help with environment issues.
- **Timing**: The suggested schedule is flexible. Some groups may need more or less time depending on their familiarity with Docker and cloud concepts.

## Acknowledgements

This training material is based on the official Kubernetes documentation and best practices as of September 2025. Please report any issues or improvements via pull requests.
