"# kubernetes-gitlab-cicd" 

Step 1

- Create Repo `"k8s-data"` on GitLab
- Create Agent on GitLab
- Install GitLab Agent on K8S Cluster
- Register Connection between K8S and GitLab

Step 2

- Create Repo `"k8s-Connection"` on GitLab
- Create K8S Manifests for our project
- Push Project’s Data on Repo `"k8s-Connection"`
- Create image and push it to GitLab Container Registry
Run final CI/CD



Step 1

Create Repo `"k8s-Connection"` on GitLab 

Create Agent on GitLab

Search in google: integrate kubernetes cluster with gitlab (https://docs.gitlab.com/ee/user/clusters/agent/) 
To connect a Kubernetes cluster to GitLab, you must first install an agent in your cluster. (https://docs.gitlab.com/ee/user/clusters/agent/install/index.html)

* In the repository, in the default branch, create an agent configuration file at: 

`.gitlab/agents/<agent-name>/config.yaml`

(This file we'll define witch group,username and repository cam acess the kubernetes cluster )

`.gitlab/agents/k8s-Connection/config.yaml`

add in file this code
```yml
ci_access:
    groups:
    - id: dev-ops-group3
    projects:
    - id: dev-ops-group3/k8s-data
```
        
Install GitLab Agent on K8S Cluster

Access GitLab: 
    Log in to your GitLab instance and navigate to the project or group where you want to integrate the Kubernetes cluster.

Enable the GitLab Agent:

- Go to Operate > Kubernetes Clusters.
- Select Connect a Cluster (Agent).
- Click on Install a new agent.

Generate an Agent Configuration file. 

This will be used later in your Kubernetes cluster.

* in gitlad repository , go to Operate > Kubernetes clusters.
* Agent Access Token : <YOUR-TOKEN>
* Helm Script :
```bash
helm repo add gitlab https://charts.gitlab.io
helm repo update
helm upgrade --install k8s-Connection gitlab/gitlab-agent \
    --namespace gitlab-agent-k8s-Connection \
    --create-namespace \
    --set config.token=<YOUR-TOKEN> \
    --set config.kasAddress=wss://kas.gitlab.com
```

Register Connection between K8S and GitLab
Exec all commands in terminal 
```bash
helm repo add gitlab https://charts.gitlab.io
helm repo update
helm upgrade --install k8s-Connection gitlab/gitlab-agent \
    --namespace gitlab-agent-k8s-Connection \
    --create-namespace \
    --set config.token=<YOUR-TOKEN> \
    --set config.kasAddress=wss://kas.gitlab.com
```

Step 2

Create Repo `"k8s-data"` on GitLab

Create K8S Manifests for our project

1- Create file .gitlab-ci.yml

2- For config this new file Search in google: gitlab container registry cicd (https://docs.gitlab.com/ee/user/packages/container_registry/build_and_push_images.html)

3- Implement Build Image

```yaml
variables:
    REGISTRY_URL: registry.gitlab.com # GitLab Container Registry URL
    REGISTRY_USERNAME: $CI_REGISTRY_USER # Predefined GitLab CI/CD variable
    REGISTRY_PASSWORD: $CI_REGISTRY_PASSWORD # Predefined GitLab CI/CD variable

stages:
    - build

build_image:
    image: docker
    stage: build
    services:
    - docker:dind
    script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
    - docker build -t $CI_REGISTRY/dev-ops-group3/k8s-data/sample:v1 .
    - docker push $CI_REGISTRY/dev-ops-group3/k8s-data/sample:v1

```
4- Create manifest file

a) Pod Manifest 

```bash
kubectl run login-app --image=registry.gitlab.com/dev-ops-group3/k8s-data/sample:v1 --dry-run=client -o yaml > pod.yaml
```

```bash 
vim pod.yaml
```
    
- Copy content in vim
    
```bash
"*y
```
    
- Result :
    
```yaml

apiVersion: v1
kind: Pod
metadata:
    labels:
    run: login-app
    name: login-app
spec:
    containers:
    - image: registry.gitlab.com/dev-ops-group3/k8s-data/sample:v1
    name: login-app
    restartPolicy: Always
    imagePullSecrets:
    - name: app-secret

```
b) Server Manifest 

```bash
kubectl create svc nodeport login-svc --tcp=80:80 --dry-run=client -o yaml > loginsvc.yaml
```
```bash 
vim loginsvc.yaml
```
- Copy content in vim

```bash
"*y
```

- Result :

```yaml
apiVersion: v1
kind: Service
metadata:
    labels:
    app: login-svc
    name: login-svc
spec:
    selector:
    run: login-app
    ports:
    - name: http
    port: 80
    targetPort: 80
    protocol: TCP
    type: NodePort

```
c) Add Create Secret Command (app-secret) in .gitlab-ci.yml 

```yaml
variables:
    KUBE_CONTEXT: dev-ops-group3/k8s-Connection:k8s-Connection
    REGISTRY_URL: registry.gitlab.com # GitLab Container Registry URL
    REGISTRY_USERNAME: $CI_REGISTRY_USER # Predefined GitLab CI/CD variable
    REGISTRY_PASSWORD: $CI_REGISTRY_PASSWORD # Predefined GitLab CI/CD variable

stages:
    - build
    - deploy

build_image:
    image: docker
    stage: build
    services:
    - docker:dind
    script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
    - docker build -t $CI_REGISTRY/dev-ops-group3/k8s-data/sample:v1 .
    - docker push $CI_REGISTRY/dev-ops-group3/k8s-data/sample:v1

deploy_project:
    stage: deploy
    image:
    name: bitnami/kubectl:latest
    entrypoint: ['']
    script:
    # Set Kubernetes context
    - kubectl config use-context $KUBE_CONTEXT
    
    # Create GitLab Container Registry secret
    - kubectl create secret docker-registry app-secret --docker-server=$REGISTRY_URL --docker-username=$REGISTRY_USERNAME --docker-password=$REGISTRY_PASSWORD --dry-run=client -o yaml | kubectl apply -f -

    # Apply Kubernetes manifests
    - kubectl apply -f $CI_PROJECT_DIR/k8s-files/
    
    # Verify deployment
    - kubectl get pods
    - kubectl get services

```
    
Push Project’s Data on Repo k8s-data

1- Create folder [k8s-files] in files [pod.yaml,loginsvc.yaml]
2- put de result of all (Create manifest file) in our respective item 

Create image and push it to GitLab Container Registry

Run final CI/CD