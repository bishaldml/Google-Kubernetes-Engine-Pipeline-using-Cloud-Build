# Google-Kubernetes-Engine-Pipeline-using-Cloud-Build
Google Kubernetes Engine Pipeline using Cloud Build

## Overview:
In this lab, we will create a CI/CD Pipeline that automatically builds a container image from committed code, stores the image in Artifact Registry, updates a k8's manifest in a Git repository, and deploys the application to GKE using that manifest.

## Objectives:
1. Create a GKE cluster.
2. Create Cloud Source Repositories.
3. Trigger Cloud Build from Cloud Source Repositories.
4. Automate tests and publish a deployable container image via Cloud Build.
5. Manage resources deployed in a GKE Cluster via Cloud Build.

## Activate Cloud Shell:
1.To list the active account name: 
```
gcloud auth list 
```
2. To list Project ID:
```
gcloud config list project
```

### Task-1: Initialize the Lab
```
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
```
```
export REGION=
```
```
gcloud config set compute/region $REGION
```
1. Enable API's for GKE, Cloud Build, Cloud Source Repositories and Container Analysis:
```
gcloud services enable container.googleapis.com \
    cloudbuild.googleapis.com \
    sourcerepo.googleapis.com \
    containeranalysis.googleapis.com
```
2. Create an Artifact Registry Docker repository:
```
gcloud artifacts repositories create my-repository \
  --repository-format=docker \
  --location=$REGION
```
3. Create a GKE Cluster:
```
  gcloud container clusters create hello-cloudbuild --num-nodes 1 --region $REGION
```
4. Configure your email and name to use git in cloud shell:
```
git config --global user.email "you@example.com"  
git config --global user.name "Your Name"
```
### Task-2: Create the Git Repository in Cloud Source Repository:
1. Create two Git Repository:
```
gcloud source repos create hello-cloudbuild-app
```
```
gcloud source repos create hello-cloudbuild-env
```
2. Clone the sample code from GitHub:
```
cd ~
git clone https://github.com/GoogleCloudPlatform/gke-gitops-tutorial-cloudbuild hello-cloudbuild-app
cd ~/hello-cloudbuild-app
```
```
export REGION="REGION"
sed -i "s/us-central1/$REGION/g" cloudbuild.yaml
sed -i "s/us-central1/$REGION/g" cloudbuild-delivery.yaml
sed -i "s/us-central1/$REGION/g" cloudbuild-trigger-cd.yaml
sed -i "s/us-central1/$REGION/g" kubernetes.yaml.tpl
```
```
PROJECT_ID=$(gcloud config get-value project)
```
```
git remote add google "https://source.developers.google.com/p/${PROJECT_ID}/r/hello-cloudbuild-app"
```
### Task-3: Create a container image with Cloud Build.
```
cd ~/hello-cloudbuild-app
```
```
cat Dockerfile
```
```
COMMIT_ID="$(git rev-parse --short=7 HEAD)"
```
```
gcloud builds submit --tag="${REGION}-docker.pkg.dev/${PROJECT_ID}/my-repository/hello-cloudbuild:${COMMIT_ID}" .
```
After the build finishes, in the Cloud console, goto Artifact Registry > Repositories to verify that your new container image is indeed available in Artifact Registry.

### Task-4: Create the CI Pipeline
```
1. In this task, we will configure Cloud Build to automatically run a small unit test, build container image, and then push it to Artifact Registry.
2. Pushing a new commit to cloud Source Repositories triggers this pipeline automatically.
3. The 'cloudbuild.yaml' file is the pipeline configuration.
```
1. In Cloud Console, Goto ```Cloud Build > Triggers```.
2. Click ```Create Trigger```
3. In ```Name field```, type ```hello-cloudbuild```
4. Under ```Event```, select ```Push to a branch```.
5. Under ```Source```, select ```hello-cloudbuild-app``` as your ```Repository``` and ```.* (any branch)``` as your ```Branch```.
6. Under ```Build configuration```, select ```Cloud Build configuration file```.
7. In the ```Cloud Build configuration file location``` field, type ```cloudbuild.yaml``` after the /.
