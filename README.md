#### DigitalOcean K8s Challenge
----

## Creating a declarative GitOps cluster managed via ArgoCD

### Prerequisites

1) [Create](https://cloud.digitalocean.com/kubernetes/clusters/new) a DO Managed Kubernetes Cluster

2) Install and configure `doctl`
2) Install and configure `kubectl`
  
### Installing ArgoCD

Understand ArgoCD architecture and terms by reading: https://argo-cd.readthedocs.io/en/stable/core_concepts/ https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/

We will choose to install all components, including UI

Setup argocd-cli
  
### Setting up the Applications in the Git Repository

* minio
* nginx
* cert-manager

* Deploy Helm charts in the declarative GitOps way
* Using App of Apps pattern to deploy multiple apps

### 
  
### Test Syncing

* Automatic sync on repository update
  
### Further reading
* Integrate ArgoCD-Notifications for alerting ArgoCD events
* Deploying Secrets declaratively using secrets management (e.g. with Bitnami Sealed Secrets)

* Install ArgoCD in High-Availability mode
* Deploying Apps to external clusters
* Using local users/SSO integration rather than default admin user
* Configure ArgoCD to listen for Git webhook events rather than polling repository
* Authenticate private Git repository using ssh key
