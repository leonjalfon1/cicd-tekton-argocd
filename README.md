# Description

# Instructions

The workshops is composed of three parts:

- Workstation Setup: configure your workstation to access your kubernetes cluster
- CI with Tekton: in this part we will use Tekton to build a docker image and store it on dockerhub
- CD with ArgoCD: in this part we will use ArgoCD to implement GitOps to manage the application deployment
- Develop and Deploy a New Version: to test the CI/CD process

# Prerequisites

- Kubernetes Cluster 1.15 or higher with RBAC enabled
- GitHub Account
- DockerHub Account

---

## Part 1: Workstation Setup

### 1) Access the Server

- Access to your workstation (the instructor will provide you with the connection details)

```
ssh sela@<your-ip>
```

### 2) Install Kubectl

- Let's install kubectl in the workstation server

```
sudo snap install kubectl --classic
```

### 3) Configure Kubectl

- Let's use gcloud CLI to configure kubectl and get access to the cluster

```
gcloud container clusters get-credentials $(hostname) --zone $(gcloud compute instances list $(hostname) --format "value(zone)") --project devops-course-architecture
```

- Test the kubectl configuration by running the following command

```
kubectl get nodes
```

---

## Part 2: CI with Tekton

In this part we will use Tekton to build a docker image and store it on dockerhub

### 1) Install Tekton

- Run the command below to install Tekton

```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

- Confirm that every component listed has the status "Running"

```
kubectl get pods --namespace tekton-pipelines
```

- Install the Tekton CLI (tkn)

```
sudo apt update;sudo apt install -y gnupg
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 3EFE0E0A2F2F60AA
echo "deb http://ppa.launchpad.net/tektoncd/cli/ubuntu eoan main"|sudo tee /etc/apt/sources.list.d/tektoncd-ubuntu-cli.list
sudo apt update && sudo apt install -y tektoncd-cli
```

- Ensure that the Tekton CLI is sucessfully installed

```
tkn version
```

### 2) Fork the Application Repository

- Browse to the link below and fork the application repository

```
https://github.com/leonjalfon1/npm-demo-app
```

<kbd><img alt="image" src="/images/image1.png" width="80%" height="60%"></kbd>

### 3) Create a Secret for DockerHub

- Create the necessary secret to access DockerHub from Tekton (secret for GitHub is not required as we're using a public repository)

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: basic-user-pass-docker
  annotations:
    tekton.dev/docker-0: https://index.docker.io # Described below
type: kubernetes.io/basic-auth
stringData:
  username: <your_docker_username>
  password: <your_docker_password>
EOF
```

- Note: Ensure that you replace the username and password with your dockerhub details

### 4) Create a Service Account 

- Create a ServiceAccount for Tekton referring to the previously created secret under the metadata

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-sa
secrets:
  - name: basic-user-pass-docker
EOF
```

### 5) Create the Pipeline Resources

- Create a PipelineResource for GitHub

```
cat << EOF | kubectl apply -f -
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: git-source
spec:
  type: git
  params:
    - name: revision
      value: master
    - name: url
      value: https://github.com/<your-github-username>/npm-demo-app
EOF
```

- Create a PipelineResource for DockerHub

```
cat << EOF | kubectl apply -f -
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: docker-target
spec:
  type: image
  params:
    - name: url
      value: index.docker.io/<your-docker-username>/npm-demo-app
EOF
```

- Note: Ensure that you replace the github and dockerhub details

### 6) Create the Tekton Task

- Create a file with the pipeline definition that uses kaniko to build the application container

```
vi tekton-task.yaml
```

```
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: build-docker-image-from-git-source
spec:
  inputs:
    resources:
      - name: git-source
        type: git
    params:
      - name: pathToDockerFile
        type: string
        description: The path to the dockerfile to build
        default: Dockerfile
      - name: pathToContext
        type: string
        description:
          The build context used by Kaniko
          (https://github.com/GoogleContainerTools/kaniko#kaniko-build-contexts)
        default: /workspace/git-source
      - name: imageTag
        description: Tag to apply to the built image
        default: "latest"
  outputs:
    resources:
      - name: builtImage
        type: image
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor
      # specifying DOCKER_CONFIG is required to allow kaniko to detect docker credential
      env:
        - name: "DOCKER_CONFIG"
          value: "/tekton/home/.docker/"
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(inputs.params.pathToDockerFile)
        - --destination=$(outputs.resources.builtImage.url):$(inputs.params.imageTag)
        - --context=$(inputs.params.pathToContext)
```

- Create the task resource

```
kubectl apply -f tekton-task.yaml
```

### 7) Run The Pipeline

- Create the TaskRun resource to execute the pipeline

```
cat << EOF | kubectl apply -f -
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: build-docker-image-from-git-source-task-run
spec:
  serviceAccountName: tekton-sa
  taskRef:
    name: build-docker-image-from-git-source
  inputs:
    resources:
      - name: git-source
        resourceRef:
          name: git-source
    params:
      - name: pathToDockerFile
        value: Dockerfile
      - name: pathToContext
        value: /workspace/git-source
      - name: imageTag
        value: v1.0
  outputs:
    resources:
      - name: builtImage
        resourceRef:
          name: docker-target
EOF
```

### 8) Track Pipeline Process

- To check the overall status use

```
tkn taskrun describe build-docker-image-from-git-source-task-run
```

- Track the pipeline logs by run (it will be available only after the pod is up and running)

```
tkn taskrun logs build-docker-image-from-git-source-task-run
```

### 9) Access the Tekton Dashboard

- To install the Tekton Dashboard run

```
kubectl apply --filename https://github.com/tektoncd/dashboard/releases/latest/download/tekton-dashboard-release.yaml
```

- Confirm that every component listed has the status "Running"

```
kubectl get pods --namespace tekton-pipelines
```

- Expose the tekton dashboard 

```
kubectl patch svc tekton-dashboard -p '{"spec": {"type": "LoadBalancer"}}' -n tekton-pipelines
```

- Retrieve the tekton dashboard external IP and browse to the portal

```
kubectl get svc tekton-dashboard -n tekton-pipelines
```
```
https://<service-external-ip>:9097
```

### 10) Inspect the Task Run in the Dashboard

- Browse to "TaskRuns"

<kbd><img alt="image" src="/images/image2.png" width="80%" height="60%"></kbd>

- Open the TaskRun Logs

<kbd><img alt="image" src="/images/image3.png" width="80%" height="60%"></kbd>

---

## Part 3: CD with ArgoCD

In this part we will use ArgoCD to implement GitOps to manage the application deployment

### 1) Install ArgoCD

- Before installing Argo let's install jq (it will useful later)

```
sudo apt update
sudo apt install -y jq
```

- Create a dedicated namespace for argocd

```
kubectl create namespace argocd
```

- Install argocd components

```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

- Install the argocd CLI

```
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')

sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64

sudo chmod +x /usr/local/bin/argocd
```

- Expose the argocd server

```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

- Retrieve the argocd external IP

```
export ARGOCD_SERVER=`kubectl get svc argocd-server -n argocd -o json | jq --raw-output .status.loadBalancer.ingress[0].ip`
echo $ARGOCD_SERVER
```

- The initial password is autogenerated with the pod name of the ArgoCD API server, retrieve it using

```
ARGO_PWD=`kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2`
echo $ARGO_PWD
```

- Login to argo using the argo CLI (using admin as username)

```
argocd login $ARGOCD_SERVER --username admin --password $ARGO_PWD --insecure
```

- Connect with ArgoCD CLI using our cluster context

```
CONTEXT_NAME=`kubectl config view -o jsonpath='{.contexts[].name}'`
argocd cluster add $CONTEXT_NAME
```

### 2) Deploy the Application

- In your repository fork edit the deployment image to set your own dockerhub username instead of "leonjalfon1" (file: /kubernets/deployment.yaml)

```
spec:
      containers:
      - image: leonjalfon1/npm-demo-app:v1.0  # <-- EDIT THIS LINE
        imagePullPolicy: Always
        name: npm-demo-app
        ports:
        - containerPort: 3000
          protocol: TCP
```

- Create a namespace to deploy the demo application

```
kubectl create namespace npm-demo-app
```

- Configure the application and link to your fork (replace the GITHUB_USERNAME):

```
argocd app create npm-demo-app --repo https://github.com/GITHUB_USERNAME/npm-demo-app.git --path kubernetes --dest-server https://kubernetes.default.svc --dest-namespace npm-demo-app
```

- Application is now setup, let’s have a look at the deployed application state:

```
argocd app get npm-demo-app
```

- We can see that the application is in an OutOfSync status since the application has not been deployed yet. Let's sync our application:

```
argocd app sync npm-demo-app
```

- Retrieve the application external IP

```
kubectl get svc npm-demo-app -n npm-demo-app
```

- Browse to the demo application

```
http://<external-service-ip>
```

<kbd><img alt="image" src="/images/image4.png" width="80%" height="60%"></kbd>

### 3) Access the ArgoCD Web Interface

- Retrieve the argocd server external ip:

```
kubectl get svc argocd-server -n argocd
```

- Browse to the web portal

```
https://<external-service-ip>
```

<kbd><img alt="image" src="/images/image5.png" width="80%" height="60%"></kbd>

- Login using "admin" as username and the following password

```
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```

- Click on the application to inspect the application details

<kbd><img alt="image" src="/images/image6.png" width="80%" height="60%"></kbd>

- Note that you can delete pods from the argocd web portal

<kbd><img alt="image" src="/images/image7.png" width="80%" height="60%"></kbd>

- Keep it open, let's add some replicas to our deployment directly on the source control. To do it click on the deployment

<kbd><img alt="image" src="/images/image8.png" width="80%" height="60%"></kbd>

- Then click on "edit" and update the replica count from "1" to "3"

<kbd><img alt="image" src="/images/image9.png" width="80%" height="60%"></kbd>

- Click save and come back to the application map

<kbd><img alt="image" src="/images/image10.png" width="80%" height="60%"></kbd>

- Note that 2 new pods were added

<kbd><img alt="image" src="/images/image11.png" width="80%" height="60%"></kbd>

- This is great but that's not the best way to work. If we follow the GitOps pattern our repository should be the single source of truth. Note that the deployment show the "OutOfSync" icon, let's click on "Sync" to align our deployment with the source code.

<kbd><img alt="image" src="/images/image12.png" width="80%" height="60%"></kbd>

- Now it's time to develop and deploy a new version!

## Part 4: Develop and Deploy a New Version

- Let's update the website title in the index.html file (line 218)

```
<div class="jumbotron text-center">
  <h1>My Demo Company</h1>  <-- UPDATE THIS LINE WITH YOUR FAVORITE TITLE
  <p>We specialize in blablabla</p> 
  <form>
    <div class="input-group">
      <input type="email" class="form-control" size="50" placeholder="Email Address" required>
      <div class="input-group-btn">
        <button type="button" class="btn btn-danger">Subscribe</button>
      </div>
    </div>
  </form>
</div>
```

- Let's access to the tekton UI to trigger a new build (retrieve the external ip with the command below)

```
kubectl get svc tekton-dashboard -n tekton-pipelines
```
```
https://<service-external-ip>:9097
```

- Browse to TaskRuns and click on "Create"

<kbd><img alt="image" src="/images/image13.png" width="80%" height="60%"></kbd>

- Set the following details and click on Create

```
Namespace: default
Task: build-docker-image-from-git-source
git-source: git-source
builtImage: docker-target
pathToDockerFile: Dockerfile
pathToContext: /workspace/git-source
imageTag: v2.0
ServiceAccount: tekton-sa
```

<kbd><img alt="image" src="/images/image14.png" width="80%" height="60%"></kbd>

- A new TaskRun will be created, let's click on it to see the logs

<kbd><img alt="image" src="/images/image15.png" width="80%" height="60%"></kbd>

- After it complete, after it finishes correctly we can edit our deployment.yaml to update the image tag from "v1.0" to "v2.0"

```
spec:
      containers:
      - image: leonjalfon1/npm-demo-app:v1.0 # <-- Update the tag from "v1.0" to "v2.0"
        imagePullPolicy: Always
        name: npm-demo-app
        ports:
        - containerPort: 3000
          protocol: TCP
```

- Finally let's sync our application from the argo web interface (retrieve the external ip with the command below)

```
kubectl get svc argocd-server -n argocd
```

```
https://<external-service-ip>
```

- You will see that the application is "OutOfSync" (if you don't see it yet, just click on "Refresh"). Then click on Sync to syncronize your application with the new deployment manifest.

<kbd><img alt="image" src="/images/image16.png" width="80%" height="60%"></kbd>

- Finally let's browse to the application to see the changes (retrieve the external ip with the command below)

```
kubectl get svc npm-demo-app -n npm-demo-app
```

```
http://<external-service-ip>
```

## What's Next?

- Can argo synchronization be automated to achieve continuous deployment instead of continuous delivery?

  - Sure, you are one button away from that:
    
    1. Select your application

    <kbd><img alt="image" src="/images/image17.png" width="80%" height="60%"></kbd>

    2. Select the Argo application

    <kbd><img alt="image" src="/images/image18.png" width="80%" height="60%"></kbd>

    3. Update the Sync Policy to "Auto-Sync"

    <kbd><img alt="image" src="/images/image19.png" width="80%" height="60%"></kbd>

- Can Tekton perform more complex builds using multiple tasks?

  - Yes, this is why there is a Tekton CRD called Pipeline which actually combine multiple tasks to perform more complex build passing the "artifacts" between them. For more info visit: https://tekton.dev/docs/pipelines/pipelines/

- Can Tekton trigger builds automatically after each commit?

  - Yes, but this requires extra configuration. You can use Tekton triggers to to map fields from an event payload into resource templates (github webhooks for example). For more info visit: https://tekton.dev/docs/triggers/