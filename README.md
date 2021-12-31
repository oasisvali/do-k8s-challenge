#### DigitalOcean K8s Challenge
----

## Creating a declarative GitOps cluster with multiple Applications managed via ArgoCD

**NOTE**:   The following steps were performed on a Ubuntu Linux 21.04 machine, but all tools used have Mac/Windows support as well

### Prerequisites

1) [Create](https://cloud.digitalocean.com/kubernetes/clusters/new) a DO Managed Kubernetes Cluster
<pic>

2) Install and configure `doctl`

    [Reference](https://docs.digitalocean.com/reference/doctl/how-to/install/)

    ```bash
    sudo snap install doctl
    # enable kubectl integration for doctl
    sudo snap connect doctl:kube-config
    ```

    Generate a DigitalOcean [Personal Access Token](https://cloud.digitalocean.com/account/api/tokens) with Read & Write access
    <pic>

    Use token to authenticate `doctl`
    ```bash
    doctl auth init --context do-k8s
    # enter token when prompted
    # switch to the auth context once authentication succeeds
    doctl auth switch --context do-k8s
    # verify
    doctl account get
    ```

3) Install and configure `kubectl`

    [Reference](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

    ```bash
    snap install kubectl --classic
    # verify
    kubectl version --client
    ```
  
    Copy the Cluster ID from the DigitalOcean Cluster Page for the newly created cluster
    <pic>

    Fetch kube context for new cluster to local kubeconfig
    ```bash
    doctl kubernetes cluster kubeconfig save <cluster-id>
    # verify
    kubectl get pods
    ```
  
### Installing ArgoCD

* [ArgoCD Architecture Reference](https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture)
* [ArgoCD Terms Reference](https://argo-cd.readthedocs.io/en/stable/core_concepts)
* [Install Reference](https://argo-cd.readthedocs.io/en/stable/getting_started/)
      
  We will install all ArgoCD components, including the UI (not just core). Also, we will install it in non-HA mode to keep the deployment lightweight. ArgoCD should be deployed in HA mode in production.

  ```bash
  kubectl create namespace argocd
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  # verify install
  kubectl get pods -n argocd
  ```
  
  Expose ArgoCD API Server via Port Forwarding
  
  ```bash
  kubectl port-forward svc/argocd-server -n argocd 8080:443
  ```
      
  The ArgoCD UI will now be available at http://localhost:8080. We can use the automatically created `admin` account to access the UI. We will be using the UI to configure ArgoCD (rather than the ArgoCD CLI). First, fetch the password for the `admin` account from the automatically created secret
  
  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
  ```
  
  Then, use the password from the above step to login to the ArgoCD UI at http://localhost:8080
  <pic>
  
  Update the password for the `admin` account
  <pic>
    
  Delete the secret containing the automatically created `admin` password
  
  ```bash
  kubectl delete secret argocd-initial-admin-secret -n argocd
  ```
        
### Setting up the Applications in the Git Repository

* [ArgoCD App-of-Apps Architecture Reference]()

For this demo, we will be deploying multiple ArgoCD `Application` resources via their Helm chart definitions. To define all the Applications under a single git repository, we will define and deploy a root ArgoCD `Application` resource. The root application will then automatically create the sub-applications as well as any standalone Kubernetes manifests in the same Git repository in the **App-of-Apps** pattern. The following ArgoCD `Application` resources will be created:

1) `root` Application - the cluster bootstrapping Application
1) [cert-manager](https://cert-manager.io/docs/) Application - deploys a helm chart
2) [ingress-nginx](https://kubernetes.github.io/ingress-nginx/) Application - deploys a helm chart
3) [minio]() Application - deploys a helm chart

Additionally, we will also create supporting resources like ingress-class, cluster-issuer, namespaces, deployments (e.g. for [sealed-secrets]()) via raw YAML definitions.

The required definitions for all the components are located in the current repository. To begin the syncing process, simply apply the `root.yaml` Application. Alternatively, the `root` Application can also be defined manually via the ArgoCD UI.

```bash
kubectl apply -n argocd -f root.yaml
```

This should deploy all the resources and their status will be visible in the ArgoCD UI
<pic>

**NOTE**: For the `cert-manager` certificates to be issued successfully, the external IP address of the `LoadBalancer` service created by `ingress-nginx` should have a Publicly propagated DNS A Record pointing to any ingress endpoint (such as `minio.oasisvali.com` in this example)

### 
  
### Test Deployment and Syncing

* Automatic sync on repository update
  
### Further reading
* Integrate ArgoCD-Notifications for alerting ArgoCD events
* Deploy multiple ArgoCD Applications using the [App-of-Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) pattern
* Install ArgoCD in High-Availability mode
* Configure ArgoCD via ArgoCD CLI
* Deploy Apps to external clusters
* Use local users/SSO integration rather than `admin` user
* Configure ArgoCD to listen for Git webhook events rather than polling repository
* Authenticate private Git repositories using ssh key/https token
