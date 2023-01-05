# First Approach on Kubernets and CD

## Project that uses ArgoCD as the GitOps cluster CD manager for a mongo database managed through mongo express.

### WSL

For this I used a WSL2 Ubuntu environment on a Windows 11 machine.

Docker, Git and Visual Studio Code need to be installed.

#### - Docker
https://docs.docker.com/desktop/windows/wsl/
Need to enable Kubernets, that will provide us with a single node cluster.
#### - Git on WSL
```
sudo apt install git
git config --global user.name "Your Name"
git config --global user.email "youremail@domain.com"
```

It is recommended to install the latest Git for Windows in order to share credentials & settings between WSL and the Windows host. Git Credential Manager is included with Git for Windows and the latest version is included in each new Git for Windows release. During the installation, you will be asked to select a credential helper, with GCM set as the default. 
https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-git 
#### - Visual Studio Code on WSL
https://code.visualstudio.com/docs/remote/wsl

### Mongo database and Mongo express
First we will use our single node cluster to develop the app that is going to be managed by ArgoCD after. It's a simple dummy application to learn about Kubernets core concepts.

For this I used the Docker Official Images for mongo and mongo-express present on DockerHub.

For the definition of the service deployment files in Kubernetes, the following components were created (in order):

- mongodb-configmap.yaml: A config map that contains all of the environment configuration variables needed to run the service (In this case only the mongo database url)
- mongo-secret.yaml: mongo root user and password, only stored with a base64 encryption.
- mongo-db-deployment.yaml: A deployment file that contains the definition of the replica set that will run within Kubernetes. The requirements of the mongo service were refined as per the standards. Also, the starting number of replicas was defined as 1.
- mongo-express-deployment.yaml: deployment file that contains the definition of the replica set that will run within Kubernetes. The requirements of the mongo-express service were refined as per the standards with a LoadBalancer and exposed externaly on port 8081. Also, the starting number of replicas was defined as 1.

This are the files that will be later tracked by ArgoCD.

### GIT
ArgoCD will track the previous files on a GitHub Repository "argocd-mongodb-test-setup"
The Repository was created on the Web UI.

Then the local files were synchronized with the Repo, on the project folder, there is other folder with the Kubernetes files called dev/:
```
git init
git add dev/
git commit
git branch -M main
git remote add origin https://github.com/fpereira1991/argocd-mongodb-test-setup.git
git push -u origin main
```

### ArgoCD
Remove any application from the Kubernetes Cluster
```
kubectl delete --all namespaces
```
Install ArgoCD
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Get port for argocd-server
```
kubectl get svc -n argocd
```
Open the port for external access on localhost:8080
```
kubectl port-forward -n argocd svc/argocd-server 8080:443
```
Access the ArgoCD Web UI:
- User: admin
- Password: ```kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo```

No Applications being tracked yet.

#### Configure ArgoCD to run the mongo application
For better understanding the configuration YAML was made manually and not on the UI

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-mongodb-test-setup
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/fpereira1991/argocd-mongodb-test-setup.git
    targetRevision: HEAD
    path: dev
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd-mongodb

  syncPolicy:
    syncOptions:
    - CreateNamespace=true

    automated:
      selfHeal: true
      prune: true
```
Then apply the application yaml on the kubernets cluster
```
kubectl apply -f application.yaml
```
Check the ArgoCD Web UI, or the componentes being installed on the namespace provided, the application must be present now, perform changes oh the YAML configuration files for the mongo application and check if argo is deploying them
