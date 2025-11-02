# EKS Deployment and CI/CD for the "Wisecow" Application

![Deployment](https://img.shields.io/badge/Deployment-Inactive-red)

**Live Deployment:** [https://wisecow-on-kubernetes.online](https://wisecow-on-kubernetes.online)

> **Note:** The live deployment at the URL above is currently inactive. The underlying AWS EKS cluster has been decommissioned to manage project costs. All deployment configurations remain available within this repository for review and educational purposes.

## 1. Project Objective

This repository contains the artifacts for the containerization, continuous integration, continuous deployment (CI/CD), and operational management of a web application on Amazon Elastic Kubernetes Service (EKS). The primary goal is to demonstrate a production-grade workflow, from source code to a secure, scalable, and highly available public-facing service.

The implementation covers four main phases:
*   **Phase 1: Containerization:** Encapsulating the application using Docker.
*   **Phase 2: Cloud-Native CI/CD:** Automating the build and deployment process with GitHub Actions and Amazon ECR.
*   **Phase 3: Infrastructure Management:** Provisioning and performing a zero-downtime version upgrade of an Amazon EKS cluster.
*   **Phase 4: Secure Public Access:** Configuring TLS termination using an Application Load Balancer (ALB) managed by the AWS Load Balancer Controller.

## 2. Technical Implementation and Challenges

This section details the technical decisions and challenges encountered during the project execution, along with the implemented solutions.

### 2.1. CI/CD Pipeline Architecture
**Initial State:** The initial CI/CD pipeline was configured for deployment to a local `minikube` cluster, which is unsuitable for a cloud-based continuous deployment model due to network isolation.

**Implemented Solution:** The pipeline was re-architected for a cloud-native environment.
*   **Container Registry:** Migrated from Docker Hub to **Amazon Elastic Container Registry (ECR)** for private, secure image storage.
*   **Authentication:** Replaced static Docker credentials with secure, short-lived tokens using the **AWS IAM** `configure-aws-credentials` and `amazon-ecr-login` GitHub Actions.
*   **Deployment Target:** Modified the deployment job to connect to the public API endpoint of the Amazon EKS cluster, enabling successful deployments from the GitHub Actions runner.

### 2.2. EKS Cluster Lifecycle Management
**Challenge:** An end-of-support notification was issued by AWS for the cluster's Kubernetes version (v1.28), necessitating a controlled, in-place upgrade.

**Implemented Solution:** A zero-downtime "blue/green" node group migration was executed.
1.  **Control Plane Upgrade:** The EKS control plane was upgraded sequentially from v1.28 to v1.30 using `eksctl upgrade cluster`.
2.  **New Node Group Provisioning:** A new EKS Managed Node Group was provisioned with a version matching the upgraded control plane.
3.  **Workload Migration:** The `kubectl drain` command was used to gracefully evict all pods from the old node group. The Kubernetes scheduler automatically rescheduled these pods onto the new, healthy nodes, ensuring service continuity. This included both the application pods and critical system components like `coredns` and the `aws-load-balancer-controller`.
4.  **Decommissioning:** The old node group was deleted using `eksctl delete nodegroup` after workload migration was verified.

### 2.3. TLS Termination and Ingress
**Challenge:** The application required secure HTTPS access from the public internet. This necessitated integration with a DNS provider, certificate authority, and an L7 load balancer.

**Implemented Solution:**
1.  **Domain and DNS:** An external domain was registered with **Namecheap**. DNS authority was then delegated to **AWS Route 53** by creating a Public Hosted Zone and updating the nameserver (NS) records at the registrar. This centralized DNS management within the AWS ecosystem.
2.  **Certificate Provisioning:** A public TLS certificate for the domain was requested from **AWS Certificate Manager (ACM)** and validated using the DNS validation method. The tight integration between Route 53 and ACM allowed for automated creation of the validation CNAME record.
3.  **Load Balancer and Ingress Configuration:**
    *   The **AWS Load Balancer Controller** was installed in the cluster using its official Helm chart.
    *   A Kubernetes `Ingress` resource was defined using `spec.ingressClassName: alb`.
    *   The Ingress was annotated with the ACM certificate's ARN and a rule to automatically redirect all HTTP traffic to HTTPS, which provisioned and configured an AWS Application Load Balancer accordingly.
    *   A final **Alias (A) record** was created in Route 53 to point the custom domain to the DNS name of the provisioned ALB.

## 3. Technology Stack

| Domain | Technology / Service | Role |
| :--- | :--- | :--- |
| **Orchestration** | Amazon EKS (v1.30) | Managed Kubernetes control plane and data plane. |
| **Containerization** | Docker | Application packaging and runtime environment. |
| **CI/CD** | GitHub Actions | Automated build, test, and deployment workflow. |
| **Container Registry**| Amazon ECR | Private, secure storage for Docker images. |
| **DNS & Domain** | Namecheap & AWS Route 53 | Domain registration and authoritative DNS hosting. |
| **Security & TLS** | AWS Certificate Manager (ACM) | Provisioning and management of public SSL/TLS certificates. |
| **Load Balancing** | AWS Load Balancer Controller | L7 Ingress management and ALB provisioning. |
| **CLI Tooling** | `kubectl` & `eksctl` | Kubernetes cluster and AWS EKS resource interaction. |

## 4. Repository Contents

This repository includes:
*   `Dockerfile`: Defines the build process for the application container image.
*   `.github/workflows/cicd-pipeline.yaml`: The GitHub Actions workflow for the CI/CD pipeline.
*   `deployment.yaml`: Kubernetes manifest for the application `Deployment`.
*   `service.yaml`: Kubernetes manifest for the application `Service`.
*   `ingress.yaml`: Kubernetes manifest for the `Ingress` resource to manage external access.

