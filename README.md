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
8. Click ```Create```.

Now we need to Push the application code to Cloud Source Repositories to trigger the CI Pipeline in Cloud Build.
```
cd ~/hello-cloudbuild-app
git add .
git commit -m "Msg-1"
git push google master
```
In Cloud Console, Goto ```Cloud Build > Dashboard```.(To view the newly build.)

### Task-5: Create the Test Environment and CD Pipeline
Cloud Build is also used for the continous delivery pipeline.

The pipeline runs each time a commit is pushed to the candidate branch of the hello-cloudbuild-env repository.

The pipeline applies the new version of the manifest to the kubernetes cluster and, if successful, copies the manifest over to the production branch.

1. Grant Cloud Build access to GKE.
```
PROJECT_NUMBER="$(gcloud projects describe ${PROJECT_ID} --format='get(projectNumber)')"
```
```
gcloud projects add-iam-policy-binding ${PROJECT_NUMBER} \
--member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
--role=roles/container.developer
```

2. Clone hello-cloudbuild-env repository and create the production branch.
```
cd ~
gcloud source repos clone hello-cloudbuild-env
cd ~/hello-cloudbuild-env
git checkout -b production
```

3. Copy the 'cloudbuild-delivery.yaml' file available in the hello-cloudbuild-app repository and commit the change.
```
cd ~/hello-cloudbuild-env
cp ~/hello-cloudbuild-app/cloudbuild-delivery.yaml ~/hello-cloudbuild-env/cloudbuild.yaml
git add .
git commit -m "Create cloudbuild.yaml for deployment"
```
4. Create a candidate branch and push both branches for them to be available in Cloud Source and Repositories:
```
git checkout -b candidate
git push origin production
git push origin candidate
```
5. Grant the Source Repository write IAM role to the cloud Build Service account for the hello-cloudbuild-env repository:
```
PROJECT_NUMBER="$(gcloud projects describe ${PROJECT_ID} \
--format='get(projectNumber)')"
cat >/tmp/hello-cloudbuild-env-policy.yaml <<EOF
bindings:
- members:
  - serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com
  role: roles/source.writer
EOF
```
```
gcloud source repos set-iam-policy \
hello-cloudbuild-env /tmp/hello-cloudbuild-env-policy.yaml
```
6. Create the Trigger for the continous delivery pipeline.
    1. In the Cloud console, go to ```Cloud Build > Triggers```.
    2. Click ```Create Trigger```.
    3. In the ```Name field```, type ```hello-cloudbuild-deploy```.
    4. Under ```Event```, select ```Push to a branch```
    5. Under ```Source```, select ```hello-cloudbuild-env``` as your ```Repository``` and ```^candidate$``` as your ```Branch```.
    6. Under ```Build configuration```, select ```Cloud Build configuration file```.
    7. In the ```Cloud Build configuration file location``` field, type ```cloudbuild.yaml``` after the /.
    8. Click ```Create```.
7. Modify the CI Pipeline to trigger the CD Pipeline.
    1. Copy the extended version of the cloudbuild.yaml file for the app repository:
    ```
    cd ~/hello-cloudbuild-app
    cp cloudbuild-trigger-cd.yaml cloudbuild.yaml
    ```
    2. Commit the modifications and push them to Cloud Source Repositories:
    ```
    cd ~/hello-cloudbuild-app
    git add cloudbuild.yaml
    git commit -m "Trigger CD pipeline"
    git push google master
    ```
    
### Task-6: Review Cloud Build Pipeline
In Cloud Console, Goto Cloud Build > Dashboard.
### Task-7: Test the complete pipeline
1. In the Cloud console, go to Kubernetes Engine > Gateways,Services & Ingress.
2. Click on the endpoint for the hello-cloudbuild service. You should see "Hello World!".
3. In Cloud Shell, replace "Hello World" with "Hello Cloud Build", both in the application and in the unit test:
```
cd ~/hello-cloudbuild-app
sed -i 's/Hello World/Hello Cloud Build/g' app.py
sed -i 's/Hello World/Hello Cloud Build/g' test_app.py
```
4. Commit and push the change to Cloud Source Repositories:
```
git add app.py test_app.py
git commit -m "Hello Cloud Build"
git push google master
```
