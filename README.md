
# Implement DevOps in Google Cloud: Challenge Lab Walkthrough

In this repo I will be explaining the thought process and steps achieved to complete the DevOps GCP Challenge Lab. [Link](https://www.cloudskillsboost.google/focuses/13287?parent=catalog) to the lab in question.

The challenge has 4 main tasks:
-   Check Jenkins pipeline has been configured.
-   Check that Jenkins has deployed a development pipeline.
-   Check that Jenkins has deployed a canary pipeline.
-   Check that Jenkins has merged a canary pipeline with production.

The Kubernetes cluster and Jenkins server are already set up from the start with some basic configuration so there is no need to install any additional components.

## Initial setup
To begin we will setup our environment to work in the corresponding zone and connect to the defined cluster:
```
gcloud config set compute/zone us-east1-b
gcloud container clusters get-credentials jenkins-cd
```
Check if the cluster connection is succesful:
```
kubectl cluster-info
```

## First task: Check Jenkins pipeline has been configured.
First step is to expose the 8080 port corresponding to Jenkins to access the web UI:
```
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
```
Next we need to get the password to access the Jenkins UI. We can retrieve it from the Kubernetes secrets:
```
printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```
Before procceeding to configure the pipeline, we need to clone the corresponding repo:
```
git clone https://source.developers.google.com/p/qwiklabs-gcp-02-0be28c218beb/r/sample-app
cd sample-app
```
Now we access Jenkins UI clicking on the ``Preview on Port 8080`` on the cloud console.
Login using the `admin` user and the password we retrieved from the Kubernetes secrets to configure the service account credentials and the multibranch pipeline.

To configure the credentials:
- In Jenkins's Dashboard, click on `Manage Jenkins` > `Manage credentials`.
- Click on ``Jenkins`` under ``Store scoped to Jenkins``.
- Click on ``Global credentials`` > ``Add credentials``.
- Select ``Google Service Account from metadata`` from the dropdown menu and then press ``OK``.

For the pipeline:
- Go back to the main dashboard page.
- Click on ``New Item`` on the left bar.
- Give a name to the item, like ``sample-app``.
- Choose the ``multibranch option`` and then press ``OK``.
- In the ``Branch Sources`` select the option ``Git``.
  - Project repository: https://source.developers.google.com/p/[PROJECT_ID]/r/sample-app
  - Credentials: Qwiklabs Service Account
- In the ``Scan Multibranch Pipeline Triggers`` select the option ``Periodically if not otherwise run``.
  - Interval: 1 minute.
- Press the ``Save`` button.

At this point the pipeline should begin to build the app. We still need to create the ``production`` namespace for the app to be deployed to as detailed in the lab instructions:
>**Tip 1** The VPC networks and Kubernetes cluster have been created for you but the Kubernetes namespaces required by your Jenkins configuration have not been created for you. You must create the production namespace that the Jenkins configuration file requires before pushing changes that will trigger a Jenkins production or canary build.

Go back to the GCP Cloud Console and enter the following command:
```
kubectl create ns production
``` 
The app should be deployed to the namespace in a few minutes. You can check your fist checkpoint at this point. To check the deployment:
```
kubectl get deployments --namespace production
```
## Task 2: Check that Jenkins has deployed a development pipeline.

From the lab instructions, it mentions the following about development branches:
> Changes to any other branch trigger a Jenkins job that deploys the application to a namespace that matches the branch name used using the `dev/*.yaml` templates.

So we start creating a new branch:
```
git checkout -b development
```
Next we edit the ``main.go`` and ``html.go`` as per instructions of the lab:
```
vim main.go
vim html.go
```
Lastly, we add our GCR credentials and push the changes to the new branch:
```
git config --global user.email "[EMAIL]"
git config --global user.name "student"
git add .
git commit -m "version update"
git push origin development
```
After 1 minute or so, Jenkins should detect the new branch and trigger a new build which wil be deployed to a namespace with the same name as the new branch. To check the deployment, use the following command:
```
kubectl get deployments --namespace development
```
Also you can check the modified app, configuring the cluster to act as a proxy:
```
kubectl proxy
curl https://localhost/api/v1/namespaces/development/services/gceme-frontend:80/proxy/version
```
Check the second checkpoint of the lab.

## Task 3: Check that Jenkins has deployed a canary pipeline.
For this task, you just have to merge the development branch into a new ``canary`` pipeline:
```
git checkout -b canary
git merge development
```
Build will start in a minute or so, and then deploy automatically to ``canary`` namespace:
```
kubectl get deployments --namespace canary
```
Check the third checkpoint of the lab.
## Task 4: Check that Jenkins has deployed a production pipeline.
This last task follows the steps as the task before but for the master branch, which will then deploy to the ``production`` namespace:
```
git checkout -b master
git merge canary
```
Build will start in a minute or so, and then deploy automatically to ``production`` namespace:
```
kubectl get deployments --namespace production
```
Check the final checkpoint of the lab.

## Final Thoughts
Since this was a paid lab and didn't tried the previous labs of the DevOps course, I didn't want to timeout searching for the commands so I wrote this guide after researching the necessary steps to complete the objectives. Even so, the checkpoint #2 was never scored properly during my run and ended up failing the lab. I don't know if I missed a step or the Qwiklabs platform just bugged out, since in other [examples](https://youtu.be/g124gdJD4so?t=2443) I've seen, it seems that it takes a few tries to get corresponding points for that objective.
