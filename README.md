#### DigitalOcean K8s Challenge
----

## Creating a declarative GitOps cluster with multiple Applications managed via ArgoCD

**NOTE**:   The following steps were performed on a Ubuntu Linux 21.04 machine, but all tools used have Mac/Windows support as well

### Prerequisites

1) [Create](https://cloud.digitalocean.com/kubernetes/clusters/new) a DO Managed Kubernetes Cluster

    ![Create DO Managed Kubernetes Cluster](images/Screenshot%20Create%20Cluster.png?raw=true)

2) Install and configure `doctl`

    [Reference](https://docs.digitalocean.com/reference/doctl/how-to/install/)

    ```bash
    sudo snap install doctl
    # enable kubectl integration for doctl
    sudo snap connect doctl:kube-config
    ```

    Generate a DigitalOcean [Personal Access Token](https://cloud.digitalocean.com/account/api/tokens) with Read & Write access
    
    ![Generate Personal Access Token](images/Screenshot%20Create%20Token.png?raw=true)

    Use the token to authenticate `doctl`
    ```bash
    doctl auth init --context do-k8s
    # enter token when prompted
    # switch to the auth context once authentication succeeds
    doctl auth switch --context do-k8s
    # verify
    doctl account get
    ```

3) Install and configure `kubectl` - [Reference](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

    ```bash
    snap install kubectl --classic
    # verify
    kubectl version --client
    ```
  
    Copy the Cluster ID from the DigitalOcean Cluster Page for the newly created cluster
    
    ![Copy Cluster ID](images/Screenshot%20Copy%20Cluster%20ID.PNG?raw=true)

    Fetch kube context for new cluster to local kubeconfig

    ```bash
    doctl kubernetes cluster kubeconfig save <cluster-id>
    # verify
    kubectl get pods
    ```
  
### Installing ArgoCD
      
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
    
The ArgoCD UI will now be available at http://localhost:8080. We can use the automatically created `admin` account to access the UI. First, fetch the password for the `admin` account from the automatically created secret

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Then, use the password from the above step to login to the ArgoCD UI

![ArgoCD UI Login](images/login.jpg?raw=true)

Update the password for the `admin` account

![ArgoCD UI Login](images/newpass.jpg?raw=true)
  
Delete the secret containing the automatically created `admin` password

```bash
kubectl delete secret argocd-initial-admin-secret -n argocd
```

* [ArgoCD Architecture Reference](https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture)
* [ArgoCD Terms Reference](https://argo-cd.readthedocs.io/en/stable/core_concepts)
* [Install Reference](https://argo-cd.readthedocs.io/en/stable/getting_started/)

### Installing sealed-secrets (Optional)

We will use [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets) to declaratively store the secrets in our GitOps repository. To use a `SealedSecret` custom resource, we need to install `kubeseal` to generate the encrypted secret and install the `sealed-secrets` controller into the cluster

```bash
curl -L https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.17.1/kubeseal-0.17.1-linux-amd64.tar.gz --output kubeseal.tar.gz
tar -xf kubeseal.tar.gz
chmod +x kubeseal

kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.17.1/controller.yaml
# verify
kubectl get pods -n kube-system

# generate secret
echo -n bar | kubectl create secret generic minio-secret -n minio --dry-run=client --from-literal access-key=`pwgen -csn 20 1` --from-literal secret-key=`pwgen -csn 20 1` -o json >minio-secret.json
./kubeseal <minio-secret.json >minio-sealed-secret.json
# minio-sealed-secret.json is added to repository
```

### Setting up the Applications in the Git Repository

For this demo, we will be deploying multiple ArgoCD `Application` resources via their Helm chart definitions. To define all the Applications under a single git repository, we will define and deploy a root ArgoCD `Application` resource. The root application will then automatically create the sub-applications as well as any standalone Kubernetes manifests in the same Git repository in the **App-of-Apps** pattern. The following ArgoCD `Application` resources will be created:

1) `root` Application - the cluster bootstrapping Application
1) [cert-manager](https://cert-manager.io/docs/) Application - deploys a helm chart
2) [ingress-nginx](https://kubernetes.github.io/ingress-nginx/) Application - deploys a helm chart
3) [minio](https://min.io/) Application - deploys a helm chart

Additionally, we will also create supporting resources like ingress-class, cluster-issuer, ingress, sealed-secret via raw YAML definitions.

The required definitions for all the components are located in the current repository. To begin the syncing process, simply apply the `root.yaml` Application. Alternatively, the `root` Application can also be defined manually via the ArgoCD UI.

```bash
kubectl apply -n argocd -f root.yaml
```

This should deploy all the resources and their status will be visible in the ArgoCD UI

![All Apps](images/allapps.jpg?raw=true)

![Root App Chart](images/rootchart.jpg?raw=true)

**NOTE**: For the `cert-manager` certificates to be issued successfully, the external IP address of the `LoadBalancer` service created by `ingress-nginx` should have a publicly propagated DNS `A` Record pointing to any ingress endpoint (such as `minio.oasisvali.com` in this example)

* [ArgoCD App-of-Apps Architecture Reference](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
  
### Test Deployment and Automatic Sync

* Access minio UI over https - this validates that all 3 apps are functioning correctly

    ![Minio UI Demo](images/Screenshot%20Minio%20Browser.PNG?raw=true)

* Automatic sync on repository update
    
    ![Automatic Sync](images/testredeploy.jpg?raw=true)
    
  
### Improvements

* Integrate [ArgoCD-Notifications](https://argocd-notifications.readthedocs.io/en/stable/) for alerting ArgoCD events

* Install ArgoCD in [High-Availability](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/) mode

* Configure ArgoCD via ArgoCD [CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/)/UI

* Deploy Apps to [External Clusters](https://argo-cd.readthedocs.io/en/stable/getting_started/#5-register-a-cluster-to-deploy-apps-to-optional)

* Use [defined users/SSO integration](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/) rather than `admin` user

* Configure ArgoCD to listen for [Git webhook](https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/) events rather than polling repository

* Authenticate [private Git repositories](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/) using ssh key/https token

* Use ArgoCD [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/) to structure and template multiple Applications
