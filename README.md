# Complete CI/CD DevOps Project 🚀
### Deploy Python Flask App on Kubernetes cluster with GitOps Approach. 

![alt text](imgs/arch.png)

---
### Workflow:
Whenever Developer writing/changes a code and push into master/main branch, GitHub Pipeline will triggered and it will test the code with Flake8 and containerized the application with new tag and push into artifacts(dockerhub) and also GitHub Actions pipeline will update Kubernetes Manifests file with new image tag then ArgoCD will look for new changes in Manifests file and will rollout new application in kubernetes. 
```
│   app.py
│   LICENSE
│   README.md
│   requirements.txt
│
├───deploy
│       deploy.yaml
│       svc.yaml
│
├───static
│       style.css
│
└───templates
        index.html
```
---
#### What you will learn:
- Git for version control
- VS Code Editor
- Docker for testing locally
- Minikube for Kubernetes 1 Node Arch. 
- GitHub for storing code
- GitHub Actions for Continous Integrity Pipeline 
- ArgoCD for Continous Deployment Pipeline
- Python Application
    - Flask Framework
    - Flake8 Module for Linting testing  
---
## Test Application Locally. 
Whenever we are creating pipeline, it is best practice to test application locally.
- Application prequisities. 
  - Python 3.9 
  - pip installed

- Clone/Fork the Repo. 
    ```
    git clone https://github.com/infosecsingh/Flask-App-GitHub-Actions-ArgoCD.git
    cd Flask-App-GitHub-Actions-ArgoCD
    ```
- Install Dependence
    ```
    pip install -r requirements.txt
    ```
- Run locally. 
    ```
    python app.py
    ```
- Access the application.
    ```
    http://localhost:5000
    ```

Note: This application is running on 5000 port, but if you want to change, you can change the port in app.py script.

---
## Containerized Application.
Before creating pipeline, test locally if your dockerfile is accurate by running container. 
We will create Dockerfile. If you don't know what is docker, please read some basic understanding about docker: https://github.com/infosecsingh/Learn-Docker

Write Dockerfile
```
# Use a stable Python version
FROM python:3.12-slim

# Set working directory
WORKDIR /app

# Copy requirements first for caching
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy rest of the app
COPY . .

# Expose Flask port
EXPOSE 5000

# Run the app
CMD ["python", "app.py"]
```
#### Lets Build and Run the Container
1. Build the Image: Run the following in the directory containing your Dockerfile:
```
docker build -t omjaju18/demo-app:v1 .
```
Note: you need to change the name of your image, according to your dockerhub username.

2. Lets create container with image.
 ```
 docker run -d -p 8080:5000 --name demoapp demoapp:v1
 ```

 If everything is working fine and you are able to access application with https://localhost:5000 then next step is to write a GitHub Pipeline.

## CI Pipeline with GitHub Actions
1. Create a directory inside your project.
    ```
    mkdir -p .github/Workflows
    ```
2. Create your first pipeline for TEST and BUILD the image. make sure it should be yaml file
    ```
  name: Test and Build

# Trigger this workflow on push to master branch
# and for any changes in any file
on:
  push:
    branches:
      - master
    paths:
      - '**/*'

jobs:
  build:
    runs-on: ubuntu-latest   # Use the latest Ubuntu runner

    steps:
      # -------------------------------
      # Step 1: Checkout code from repo
      # -------------------------------
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          persist-credentials: false  
          # Set to false because we will push using our own PAT.
          # If true, GitHub Actions uses the github-actions bot token instead.

      # -------------------------------
      # Step 2: Set up Python environment
      # -------------------------------
      - name: Setup Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.9'  # Use Python 3.9 for consistency

      # -------------------------------
      # Step 3: Setup Docker Buildx
      # -------------------------------
      - name: Docker Setup
        uses: docker/setup-buildx-action@v2
        # Allows advanced Docker builds, multi-platform builds, and caching

      # -------------------------------
      # Step 4: Install dependencies
      # -------------------------------
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip   # Upgrade pip to latest version
          pip install -r requirements.txt      # Install Python dependencies
          pip install flake8                    # Install flake8 for linting

      # -------------------------------
      # Step 5: Run Linting tests
      # -------------------------------
      - name: Run Linting tests
        run: |
          flake8 --ignore=E501,F401 .  # Check code style and imports
          # Ignore long line errors (E501) and unused imports (F401)

      # -------------------------------
      # Step 6: Docker login
      # -------------------------------
      - name: Docker Credentials
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}  # DockerHub username from secrets
          password: ${{ secrets.DOCKER_PASSWORD }}  # DockerHub password/token from secrets

      # -------------------------------
      # Step 7: Create version tag for Docker image
      # -------------------------------
      - name: Docker tag
        id: version
        run: |
          VERSION=v$(date +"%Y%m%d%H%M%S")  # Create timestamped version, e.g., v20260103143001
          echo "VERSION=$VERSION" >> $GITHUB_ENV  # Save to GitHub environment variable

      # -------------------------------
      # Step 8: Build Docker image
      # -------------------------------
      - name: Build Docker Image
        run: |
          docker build . -t omjaju18/weather-app:${{ env.VERSION }}  
          # Build Docker image and tag with version

      # -------------------------------
      # Step 9: Push Docker image to DockerHub
      # -------------------------------
      - name: Push Docker Image
        run: |
          docker push omjaju18/weather-app:${{ env.VERSION }}  
          # Push the built image to your DockerHub repository

      # -------------------------------
      # Step 10: Update Kubernetes manifests
      # -------------------------------
      - name: Update K8s Manifests
        run: |
          cat deploy/deploy.yaml  # Show current manifest
          # Update the container image in deploy.yaml to new version
          sed -i "s|image: omjaju18/weather-app:.*|image: omjaju18/weather-app:${{ env.VERSION }}|g" deploy/deploy.yaml
          cat deploy/deploy.yaml  # Show updated manifest

      # -------------------------------
      # Step 11: Commit and push updated manifest
      # -------------------------------
      - name: Commit the changes
        run: |
          git config user.name "omjaju18"               # Set Git commit username
          git config user.email "omjaju03@gmail.com"    # Set Git commit email
          git add deploy/deploy.yaml                     # Stage the updated file
          git commit -m "Update deploy.yaml with new image version - ${{ env.VERSION }}" || echo "No changes to commit"
          # Commit changes, or ignore if no changes
          # Set the remote URL with your PAT for authentication
          git remote set-url origin https://omjaju18:${{ secrets.TOKEN }}@github.com/omjaju18/Flask-App-GitHub-Actions-ArgoCD.git
          git push origin master  # Push changes to master branch

    ```

1. Make sure setup your docker Personal Access token into github repo. 

## Setup ArgoCD in Minikube

Note: You can setup Argo CD in any cluster, instructions are same. 

- First install Minikube:
    Installation guide for installing Minikube. 
    [Minikube.sigs.k8s.io](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download)

---
- Install Argo CD
    ```
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml 
    ```

- Verify if ArgoCD is running:
    ```
    kubectl get all -n argocd
    ```
    Output
    ```    
        NAME                                                    READY   STATUS    RESTARTS       AGE
    pod/argocd-application-controller-0                     1/1     Running   0              2m57s
    pod/argocd-applicationset-controller-64f6bd6456-6jv2z   1/1     Running   0              2m57s
    pod/argocd-dex-server-5fdcd9df8b-6ctpr                  1/1     Running   1 (2m2s ago)   2m57s
    pod/argocd-notifications-controller-778495d96f-rhj9k    1/1     Running   0              2m57s
    pod/argocd-redis-69fd8bd669-5cwkf                       1/1     Running   0              2m57s
    pod/argocd-repo-server-75567c944-mth62                  1/1     Running   0              2m57s
    pod/argocd-server-5c768cdd96-6rpdp                      1/1     Running   0              2m57s

    NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
    service/argocd-applicationset-controller          ClusterIP   10.96.214.139    <none>        7000/TCP,8080/TCP            2m57s
    service/argocd-dex-server                         ClusterIP   10.105.242.131   <none>        5556/TCP,5557/TCP,5558/TCP   2m57s
    service/argocd-metrics                            ClusterIP   10.108.182.252   <none>        8082/TCP                     2m57s
    service/argocd-notifications-controller-metrics   ClusterIP   10.106.4.82      <none>        9001/TCP                     2m57s
    service/argocd-redis                              ClusterIP   10.98.222.183    <none>        6379/TCP                     2m57s
    service/argocd-repo-server                        ClusterIP   10.103.237.141   <none>        8081/TCP,8084/TCP            2m57s
    service/argocd-server                             ClusterIP   10.107.245.182   <none>        80/TCP,443/TCP               2m57s
    service/argocd-server-metrics                     ClusterIP   10.108.248.213   <none>        8083/TCP                     2m57s

    NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/argocd-applicationset-controller   1/1     1            1           2m57s
    deployment.apps/argocd-dex-server                  1/1     1            1           2m57s
    deployment.apps/argocd-notifications-controller    1/1     1            1           2m57s
    deployment.apps/argocd-redis                       1/1     1            1           2m57s
    deployment.apps/argocd-repo-server                 1/1     1            1           2m57s
    deployment.apps/argocd-server                      1/1     1            1           2m57s

    NAME                                                          DESIRED   CURRENT   READY   AGE
    replicaset.apps/argocd-applicationset-controller-64f6bd6456   1         1         1       2m57s
    replicaset.apps/argocd-dex-server-5fdcd9df8b                  1         1         1       2m57s
    replicaset.apps/argocd-notifications-controller-778495d96f    1         1         1       2m57s
    replicaset.apps/argocd-redis-69fd8bd669                       1         1         1       2m57s
    replicaset.apps/argocd-repo-server-75567c944                  1         1         1       2m57s
    replicaset.apps/argocd-server-5c768cdd96                      1         1         1       2m57s

    NAME                                             READY   AGE
    statefulset.apps/argocd-application-controller   1/1     2m57s
    ```

---
- Access ArgoCD With configuring NodePort 
    ```
    kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
    ```
- Verify if ArgoCD server running as NodePort.
   ```
   kubectl get svc -n argocd
   ``` 
   Output
   ```
   NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
    argocd-applicationset-controller          ClusterIP   10.96.214.139    <none>        7000/TCP,8080/TCP            7m17s
    argocd-dex-server                         ClusterIP   10.105.242.131   <none>        5556/TCP,5557/TCP,5558/TCP   7m17s
    argocd-metrics                            ClusterIP   10.108.182.252   <none>        8082/TCP                     7m17s
    argocd-notifications-controller-metrics   ClusterIP   10.106.4.82      <none>        9001/TCP                     7m17s
    argocd-redis                              ClusterIP   10.98.222.183    <none>        6379/TCP                     7m17s
    argocd-repo-server                        ClusterIP   10.103.237.141   <none>        8081/TCP,8084/TCP            7m17s
    argocd-server                             NodePort    10.107.245.182   <none>        80:30692/TCP,443:31365/TCP   7m17s
    argocd-server-metrics                     ClusterIP   10.108.248.213   <none>        8083/TCP                     7m17s
   ```
- Grab ArgoCD secret for accessing UI
   ```
   kubectl get secrets -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
   ```

- Start Minkube Service. 
   ```
    minikube service argocd-server -n argocd
    ```
    Output
    ```
    |-----------|---------------|-------------|-----------------------------|
    | NAMESPACE |     NAME      | TARGET PORT |             URL             |
    |-----------|---------------|-------------|-----------------------------|
    | argocd    | argocd-server | http/80     | http://172.29.213.129:30692 |
    |           |               | https/443   | http://172.29.213.129:31365 |
    |-----------|---------------|-------------|-----------------------------|
    [argocd argocd-server http/80
    https/443 http://172.29.213.129:30692
    http://172.29.213.129:31365]

   ```
   Username: admin
   password: secret(please check above command)

   ![alt text](imgs/ui.png)

---
Setup our Continous deployment. 

- Select New App.
![alt text](imgs/setting1.png)
![alt text](imgs/setting2.png)
---
- Syncing your manifests files:
![alt text](imgs/sync.png)
---

- Successfully Deployed our app:
![alt text](imgs/deployed.png)
---
Access Application with below command.
```
minikube service list
```
Output
```
|-------------|-----------------------------------------|--------------|-----------------------------|
|  NAMESPACE  |                  NAME                   | TARGET PORT  |             URL             |
|-------------|-----------------------------------------|--------------|-----------------------------|
| argocd      | argocd-applicationset-controller        | No node port |                             |
| argocd      | argocd-dex-server                       | No node port |                             |
| argocd      | argocd-metrics                          | No node port |                             |
| argocd      | argocd-notifications-controller-metrics | No node port |                             |
| argocd      | argocd-redis                            | No node port |                             |
| argocd      | argocd-repo-server                      | No node port |                             |
| argocd      | argocd-server                           | http/80      | http://172.29.213.129:30692 |
|             |                                         | https/443    | http://172.29.213.129:31365 |
| argocd      | argocd-server-metrics                   | No node port |                             |
| default     | kubernetes                              | No node port |                             |
| default     | weather-check-service                   |         5000 | http://172.29.213.129:30008 |
| kube-system | kube-dns                                | No node port |                             |
|-------------|-----------------------------------------|--------------|-----------------------------|
```
---
Application running on http://172.29.213.129:30008

![alt text](imgs/application.png)
