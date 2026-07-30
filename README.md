# Amazon EKS CI/CD Deployment

## Project Overview

This project demonstrates the deployment of a containerized Node.js application with a MongoDB backend on **Amazon Elastic Kubernetes Service (EKS)** using a complete CI/CD pipeline. The deployment leverages **GitHub Actions**, **Amazon Elastic Container Registry (ECR)**, **Amazon EKS**, **Amazon Elastic Block Store (EBS)**, and an **internet-facing Network Load Balancer (NLB)**.

The solution follows cloud-native best practices by providing:

- Automated CI/CD using GitHub Actions
- Secure authentication with AWS using OIDC
- Container image management with Amazon ECR
- Kubernetes orchestration using Amazon EKS
- Persistent storage using the EBS CSI Driver
- Automatic traffic distribution through an internet-facing Network Load Balancer
- Automatic application scaling using the Horizontal Pod Autoscaler (HPA)

---

# Architecture Diagram

```text
                                   GitHub
                                      │
                           Code Push (main branch)
                                      │
                                      ▼
                           GitHub Actions Workflow
                    (Build → Push → Deploy using OIDC)
                                      │
                 AssumeRoleWithWebIdentity (OIDC)
                                      │
                                      ▼
                               AWS IAM Role
                           (GitHub Actions Role)
                                      │
             ┌────────────────────────┴────────────────────────┐
             │                                                 │
             ▼                                                 ▼
       Amazon ECR                                   Amazon EKS Cluster
 (Docker Image Repository)                   (Managed Kubernetes Control Plane)
             │                                                 │
             │ Docker Image                                    │ kubectl apply
             ▼                                                 ▼
                                          ┌─────────────────────────────┐
                                          │        Kubernetes           │
                                          │                             │
                                          │ Deployment (2 Replicas)     │
                                          │        │                    │
                                          │        ▼                    │
                                          │  Namegen Pods               │
                                          │  (Node.js Application)      │
                                          │        │                    │
                                          │        ▼                    │
                                          │     Service                 │
                                          │ (LoadBalancer)              │
                                          └─────────┬───────────────────┘
                                                    │
                                         Internet-facing NLB
                                                    │
                                                    ▼
                                                 End Users
                                                    │
                                                    ▼
                                              HTTP Requests
                                                    │
                                                    ▼
                                            Node.js Application
                                                    │
                                             MongoDB Database
                                                    │
                                                    ▼
                                              StatefulSet
                                                    │
                                           PersistentVolumeClaim
                                                    │
                                                    ▼
                                             EBS CSI Driver
                                                    │
                                                    ▼
                                              Amazon EBS

                          Horizontal Pod Autoscaler (HPA)
                                   │
                      Monitors Resource Utilization
                                   │
                     Scales the Deployment Automatically
```

---

# AWS Services and Kubernetes Components

## GitHub

GitHub hosts the application's source code, Kubernetes manifests, Dockerfile, and GitHub Actions workflow.

Every push to the `main` branch automatically starts the deployment pipeline.

---

## GitHub Actions

GitHub Actions implements the Continuous Integration and Continuous Deployment (CI/CD) pipeline.

The workflow performs the following tasks:

- Checks out the application source code
- Authenticates to AWS using OIDC
- Builds the Docker image
- Pushes the Docker image to Amazon ECR
- Configures Kubernetes access
- Deploys the updated application to Amazon EKS

---

## OpenID Connect (OIDC)

OIDC enables GitHub Actions to authenticate with AWS without storing long-term AWS access keys.

The authentication flow is:

```
GitHub Actions
        │
OIDC Identity Token
        │
AWS Security Token Service (STS)
        │
Temporary AWS Credentials
```

### Benefits

- Eliminates long-term AWS credentials
- Improves security
- Uses temporary credentials
- Simplifies credential management

---

## AWS IAM Role

An IAM role is assumed by GitHub Actions through OIDC.

The role grants permissions required to:

- Push Docker images to Amazon ECR
- Access the Amazon EKS cluster
- Deploy Kubernetes resources

---

## Amazon Elastic Container Registry (ECR)

Amazon ECR is a managed Docker image repository.

After the application image is built, it is pushed to Amazon ECR.

Whenever Kubernetes creates a new Pod, the image is automatically pulled from ECR.

Deployment flow:

```
Source Code
      │
docker build
      │
Docker Image
      │
docker push
      ▼
Amazon ECR
```

---

## Amazon Elastic Kubernetes Service (EKS)

Amazon EKS is AWS's managed Kubernetes service.

AWS manages:

- Kubernetes control plane
- API Server
- etcd database
- Cluster availability
- Kubernetes upgrades

The application deployment manages:

- Deployments
- Pods
- Services
- ConfigMaps
- Secrets

---

## Kubernetes Deployment

A Deployment ensures that the desired number of application replicas is always running.

Responsibilities include:

- Creating Pods
- Replacing failed Pods
- Rolling updates
- Self-healing

The deployment maintains multiple replicas of the Node.js application for high availability.

---

## Kubernetes Pods

A Pod is the smallest deployable unit in Kubernetes.

Each Pod contains:

- Node.js application
- Express web server
- REST API
- Static web interface

Pods are ephemeral and are automatically recreated if a failure occurs.

---

## Kubernetes Service

Pods receive dynamic IP addresses.

A Kubernetes Service provides:

- Stable virtual IP
- Stable DNS name
- Internal load balancing

The Service is configured as:

```yaml
Type: LoadBalancer
Port: 80
TargetPort: 8080
```

Traffic flow:

```
Internet
     │
Network Load Balancer
     │
Kubernetes Service
     │
Application Pods
```

---

## Network Load Balancer (NLB)

An AWS Network Load Balancer is automatically provisioned when a Kubernetes Service of type `LoadBalancer` is created.

Responsibilities include:

- Public application endpoint
- Health checks
- Traffic distribution
- High availability

Initially, the load balancer was created as an **internal** NLB.

Adding the following annotation created a public load balancer:

```yaml
service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
```

---

## Target Group

The Network Load Balancer forwards requests to a Target Group containing the application Pods.

Initially, the health checks failed because the application listened on a Unix socket rather than TCP port 8080 due to an incorrect environment variable:

```text
SERVER_PORT=<optional_server_port_other_than_the_default_8080>
```

After changing the value to:

```text
SERVER_PORT=8080
```

the targets became healthy and the application became accessible.

---

## Horizontal Pod Autoscaler (HPA)

The Horizontal Pod Autoscaler automatically adjusts the number of Pods according to resource utilization.

Typical scaling process:

```
2 Pods
   │
High CPU Usage
   │
HPA
   │
Scale Out
   ▼
4 Pods
```

When utilization decreases:

```
4 Pods
   │
Low CPU Usage
   │
Scale In
   ▼
2 Pods
```

### Benefits

- Automatic scaling
- High availability
- Efficient resource utilization
- Reduced infrastructure costs

---

## MongoDB StatefulSet

MongoDB is deployed using a StatefulSet because databases require persistent identity and storage.

StatefulSets provide:

- Stable Pod names
- Stable hostnames
- Persistent storage associations

Unlike Deployments, StatefulSets preserve storage across Pod recreation.

---

## Persistent Volume Claim (PVC)

A Persistent Volume Claim requests durable storage for MongoDB.

Instead of storing database files inside the Pod filesystem, MongoDB stores data on a Persistent Volume.

This ensures data persistence even if Pods are recreated.

---

## Amazon EBS CSI Driver

The Amazon EBS CSI Driver enables Kubernetes to provision and manage Amazon Elastic Block Store (EBS) volumes.

Storage provisioning flow:

```
PersistentVolumeClaim
        │
        ▼
Amazon EBS CSI Driver
        │
        ▼
Amazon EBS Volume
```

The CSI Driver automatically creates, attaches, mounts, and manages EBS volumes for Kubernetes workloads.

---

## Amazon Elastic Block Store (EBS)

Amazon EBS provides durable block storage for the MongoDB database.

Features include:

- Persistent storage
- High performance
- Automatic attachment to worker nodes
- Data durability across Pod restarts

---

# Application Request Flow

```
End User
     │
Internet
     │
Internet-facing Network Load Balancer
     │
Kubernetes Service
     │
Application Pod
     │
MongoDB
     │
Persistent Volume Claim
     │
Amazon EBS
```

---

# CI/CD Workflow

```
Developer
     │
Git Push
     │
     ▼
GitHub Repository
     │
     ▼
GitHub Actions
     │
OIDC Authentication
     │
     ▼
AWS IAM Role
     │
 ┌───┴──────────────┐
 │                  │
 ▼                  ▼
Amazon ECR      Amazon EKS
(Image)          (Deployment)
                     │
                     ▼
              Rolling Update
                     │
                     ▼
              Application Pods
                     │
                     ▼
      Internet-facing Network Load Balancer
                     │
                     ▼
                 End Users
```

---

# Conclusion

This project demonstrates the deployment of a cloud-native application on Amazon EKS using a secure and automated CI/CD pipeline.

The implemented architecture combines:

- **GitHub Actions** for CI/CD automation
- **OIDC** for secure AWS authentication
- **Amazon ECR** for Docker image management
- **Amazon EKS** for Kubernetes orchestration
- **Amazon EBS CSI Driver** for dynamic storage provisioning
- **Amazon EBS** for persistent database storage
- **Network Load Balancer** for external traffic distribution
- **Horizontal Pod Autoscaler** for automatic application scaling

The resulting solution provides an automated, scalable, secure, and highly available deployment architecture that follows modern cloud-native best practices.
