# Thanos Multi-Cluster Monitoring on AWS EKS

## Overview

This repository provides a scalable solution for monitoring two Amazon EKS clusters (`eks-cluster-1`, `eks-cluster-2`) in `us-west-2` using Thanos, Prometheus, and Grafana. It aggregates real-time metrics, stores historical data in Amazon S3 (`thanos-poc-storage27`), and visualizes insights via Grafana dashboards. Deployed with Helm (`kube-prometheus-stack` v58.2.0), it ensures security via AWS IAM, OIDC, and Kubernetes service accounts, addressing challenges like VPC isolation and CIDR overlap through public LoadBalancer exposure.

## Architecture

### Components
- **Prometheus + Thanos Sidecar:** Collects metrics in both clusters, uploads to S3 every 30 minutes, and serves real-time data via gRPC.
- **Thanos Querier:** In `eks-cluster-1`, aggregates real-time data from both Sidecars and historical S3 data.
- **Grafana:** Visualizes metrics in `eks-cluster-1`, integrated with the Querier.
- **Amazon S3:** Stores historical blocks for long-term retention.
- **Public LoadBalancer:** Exposes `eks-cluster-2`’s Sidecar for cross-cluster access.


## Prerequisites
- AWS EKS, IAM, S3, and VPC in `us-west-2`.
- `kubectl`, Helm 3.x, and AWS CLI configured.
- EKS clusters `eks-cluster-1` and `eks-cluster-2` with nodes, OIDC/IAM setup.

## Installation Steps
1. **Create EKS Clusters:**
   ```bash
   aws eks create-cluster --name eks-cluster-1 --region us-west-2 --kubernetes-version 1.27
   aws eks create-cluster --name eks-cluster-2 --region us-west-2 --kubernetes-version 1.27
   aws eks update-kubeconfig --region us-west-2 --name eks-cluster-1
   aws eks update-kubeconfig --region us-west-2 --name eks-cluster-2

Why Needed: EKS clusters are the foundation for running Kubernetes workloads. Creating eks-cluster-1 and eks-cluster-2 provides isolated environments for multi-cluster monitoring, hosting Prometheus, Thanos, and Grafana. update-kubeconfig configures kubectl access, enabling cluster management and deployment.

Set Up IAM Roles and Policies:
Use eksctl or AWS IAM to create roles (eksctl-eks-cluster-1-nodegroup-ng--NodeInstanceRole-*, eksctl-eks-cluster-2-nodegroup-ng--NodeInstanceRole-*) with AmazonEKSWorkerNodePolicy, AmazonEKS_CNI_Policy, and AmazonSSMManagedInstanceCore.

Create S3 policy: aws iam create-policy --policy-name ThanosS3Access --policy-document file://s3-policy.json (custom for thanos-poc-storage27).
Why Needed: IAM roles grant nodes permissions for Kubernetes operations (e.g., joining clusters, networking). The S3 policy ensures Thanos Sidecars can upload historical data to thanos-poc-storage27, maintaining security and access control via AWS IAM.

Configure OIDC and Service Accounts:
------------------------------------
aws eks create-oidc-provider --cluster-name eks-cluster-1 --region us-west-2
aws eks create-oidc-provider --cluster-name eks-cluster-2 --region us-west-2
kubectl apply -f cluster-1/thanos-object-storage.yaml -n monitoring

-> Why Needed: OIDC providers enable Kubernetes service accounts to assume IAM roles securely, avoiding node-level credentials. This step secures S3 access for Thanos Sidecar via the thanos service account, ensuring encrypted, role-based authentication for historical data uploads.


Install Helm and Deploy kube-prometheus-stack:
----------------------------------------------
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 && chmod +x get_helm.sh && ./get_helm.sh
helm upgrade prometheus prometheus-community/kube-prometheus-stack --version 58.2.0 -n monitoring -f cluster-1/values-cluster1.yaml
helm upgrade prometheus prometheus-community/kube-prometheus-stack --version 58.2.0 -n monitoring -f cluster-2/values-cluster2.yaml

-> Why Needed: Helm simplifies Kubernetes deployments. Installing Helm enables chart management, while kube-prometheus-stack deploys Prometheus, Thanos, Grafana, and more. Custom values-clusterX.yaml files configure multi-cluster monitoring, S3 uploads, and real-time data, ensuring scalability and consistency.

Expose eks-cluster-2 Thanos Sidecar Publicly:
----------------------------------------------
kubectl apply -f cluster-2/thanos-sidecar-service-cluster2.yaml -n monitoring
Why Needed: VPC isolation and CIDR overlap prevent direct gRPC access between clusters. This LoadBalancer exposes eks-cluster-2’s Thanos Sidecar, allowing eks-cluster-1’s Querier to fetch real-time data, enabling immediate multi-cluster aggregation.

Configure Thanos Querier in eks-cluster-1:
-------------------------------------------
kubectl apply -f cluster-1/thanos-querier.yaml -n monitoring
kubectl delete pod -n monitoring -l app=thanos-querier
Why Needed: The Thanos Querier in eks-cluster-1 aggregates real-time data from both Sidecars and historical data from S3, centralizing monitoring. Applying the manifest and restarting ensures the Querier reflects updated configurations, supporting Grafana visualization.


Verify Data and Setup:
------------------------
kubectl port-forward -n monitoring svc/thanos-querier 9091:9091
curl http://localhost:9091/api/v1/query?query=up
aws s3 ls s3://thanos-poc-storage27/ --recursive
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
Why Needed: Verification ensures the setup works. Port-forwarding and querying the Querier confirm real-time data from both clusters. S3 checks validate historical uploads after 30 minutes, while Grafana access enables dashboard creation, ensuring end-to-end functionality.

Configuration Files
---------------------
cluster-1/values-cluster1.yaml: eks-cluster-1 settings (Querier, Sidecar, Grafana).
cluster-2/values-cluster2.yaml: eks-cluster-2 settings (Sidecar only).
cluster-1/thanos-querier.yaml: Thanos Querier for eks-cluster-1.
cluster-2/thanos-sidecar-service-cluster2.yaml: Public LoadBalancer for eks-cluster-2 Sidecar.
