# Unicodex Tutorial

*This tutorial was originally designed for PyCon 2020, but may be used at other events in the future. If you are following this event from PyCon 2020, ensure you choose the "pycon2020" options. This tutorial is a living entity, but the "pycon2020" version will (to the best effort of the author) match the PyCon 2020 Tutorial recording.*

PyCon 2020 Tutorial: Deploying Django on Serverless Infrastructure

 * [Description](https://us.pycon.org/2020/schedule/presentation/62/), [Recording](#), [Slides](slides.podium)

# Contents

1. [Introduction](#introduction)
1. [Demo Application](#demo-application)
1. [Setting up your terminal](#setting-up-your-terminal)
1. [Backing services](#backing-services)
1. [Build, migrate, and deploy](#build-migrate-and-deploy)
1. [Automate deployment](#automate-deployment)
1. [Cleanup](#cleanup)

# Introduction

In this tutorial, you will deploy a Django application on [Google Cloud](https://cloud.google.com/), running the service on [Cloud Run](https://http://cloud.run/), connecting to a managed database ([Cloud SQL](https://cloud.google.com/sql)), using object storage ([Cloud Storage](http://cloud.google.com/storage)), using environment secrets ([Secret Manager](https://cloud.google.com/secret-manager)), and use deployment pipelines ([Cloud Build](https://cloud.google.com/cloud-build)). 

This tutorial is designed to be accompanied by an instructor or video recording. Otherwise, you should read through the [self-paced text-only tutorial](https://github.com/GoogleCloudPlatform/django-demo-app-unicodex). 

For this tutorial, you will need a Google Cloud account. If you don't already have account or a credentials haven't been provided for you, you can register for a Google Cloud account by visiting [https://console.cloud.google.com/getting-started](https://console.cloud.google.com/getting-started). 

It is recommended that you create a new project for this tutorial. This will allow you to keep your tutorial work isolated, and allow for easier cleanup after you have finished with your project. Many of the components involved have an ongoing cost, so it is recommended you [cleanup](#cleanup) once you are done. 

# Demo Application

In this section, we will deploy a local version of the demo application, to see how it runs. 

You will need to install the following components for your operating system: 

* [Docker Desktop](https://www.docker.com/products/docker-desktop)
* [Docker Compose](https://docs.docker.com/compose/install/#install-compose)

## Download the sample application

Download a [copy of the source code for the demo application](https://github.com/GoogleCloudPlatform/django-demo-app-unicodex/releases/tag/pycon2020), an application called "unicodex". Extract the contents of the zipfile into a directory, and open your terminal to that folder. 

```
curl https://github.com/GoogleCloudPlatform/django-demo-app-unicodex/archive/pycon2020.zip -Lo unicodex-tutorial.zip
unzip unicodex-tutorial.zip
cd django-unicodex-tutorial-pycon2020
```

The current folder will contain many files, including a `Dockerfile` and `docker-compose.yml` file, and many folders, including a `unicodex` folder. 

## Locally deploy the sample application

The `docker-compose.yml` file contains the configurations required to run a web container against a postgres container. Take the time to look at this file. You can choose to change the default values for the admin password and secret key, but that is not nessessary for this test. 

Now, run the Docker Compose commands to build the image, migrate the data, and run the application. In one terminal: 

```shell
docker-compose up db
```

In another terminal:  

```shell
docker-compose build
docker-compose run --rm web python manage.py migrate
docker-compose run --rm web python manage.py loaddata sampledata
docker-compose up web
```

You can now see how the application looks and works running on your local machine by visiting [http://localhost:8080](http://localhost:8080)

Try the website out, and log into the Django admin at [http://localhost:8080/admin](http://localhost:8080) using the username and password listed in the `docker-compose.yml` file under the environment variables `SUPERUSER` and `SUPERPASS`, respectively.

Once you have finished testing out the application, stop the `docker-compose` processes with <kbd>Ctrl</kbd> + <kbd>c</kbd> (<kbd>control </kbd> + <kbd>c</kbd> on Mac keyboards).

ℹ️ After the initial database migration, your local installation can be started again using `docker-compose up`. 


# Manual Deployment

You will now go through the process of deploying this demo application to Google Cloud. 

## Setting up your project

You will need a Google Cloud account to complete this tutorial. 

Sign into the [Cloud Console](http://console.cloud.google.com/) and create a new project. (If you don't already have a Gmail or G Suite account, you must [create one](https://accounts.google.com/SignUp).) 

Next, you'll need to [enable billing](https://console.cloud.google.com/billing) in Cloud Console in order to use Google Cloud resources. Running through this codelab shouldn't cost you more than a few dollars, but it could be more if you decide to use more resources or if you leave them running. New users of Google Cloud are eligible for a [\$300 free trial](http://cloud.google.com/free).

### Setting up your shell

This tutorial can be completed using the [Google Cloud Shell](https://cloud.google.com/shell). This environment includes all of the tools we will need for the rest of this tutorial, including the Google Cloud command-line tool [`gcloud`](https://cloud.google.com/sdk/gcloud). 

In the Google Cloud console, select your project, and click the "Activate Cloud Shell" icon on the top right of the screen, immediately right of the "Search resources and products" dialog. Ensure your Cloud Shell shows your project in the prompt: 

```
yourusername@cloudshell:~ (YourProjectID)$ 
```

Confirm the Project ID has been set in `gcloud`: 

```shell
gcloud config get-value project
```

Set your Project ID as an environment variable to easily use the later commands: 

```shell
export PROJECT_ID=$(gcloud config get-value project)
echo $PROJECT_ID
``` 

ℹ️ If you prefer, you can install [`gcloud`](https://cloud.google.com/sdk/docs/#install_the_latest_cloud_tools_version_cloudsdk_current_version) on your local machine.  Ensure you have configured `gcloud` to reference the project you created for this tutorial. 

```shell
export PROJECT_ID=YourProjectID
gcloud config set project $PROJECT_ID
```

### Set your region

Select the region that your components will be deployed within: 

```shell
export REGION=us-central1
gcloud config set run/region $REGION
```

ℹ️ For this tutorial, we recommend using `us-central1`, `europe-north1`, or `asia-northeast1`, but you can choose [any region Cloud Run (fully managed) is supported](https://cloud.google.com/run/docs/locations#managed). 


## Backing services

In this section, you will setup the backing services that your hosted application will use. You will setup the Google Cloud project for this tutorial, create a database, create a storage bucket, and create a number of secrets. 

### Enable the Cloud APIs

Enable the Cloud APIs for the components that will be used:

```shell
gcloud services enable \
  run.googleapis.com \
  iam.googleapis.com \
  compute.googleapis.com \
  sql-component.googleapis.com \
  sqladmin.googleapis.com \
  cloudbuild.googleapis.com \
  cloudkms.googleapis.com \
  cloudresourcemanager.googleapis.com \
  secretmanager.googleapis.com
```

This command will take a minute to complete, and should produce a successful message similar to this one:

```
Operation "operations/acf.cc11852d-40af-47ad-9d59-477a12847c9e" finished successfully.
```

### Create and configure service accounts

Create a service account, the entity that will have authorisation to administrator the backing services, and perform automation tasks: 

```shell
gcloud iam service-accounts create unicodex-sa --display-name "unicodex service account"
```

Store a copy of the service account's identifying email address (this will be used later): 

```shell
export UNICODEX_SA=unicodex-sa@${PROJECT_ID}.iam.gserviceaccount.com
```

Grant the service account administrator access to administer Cloud Run, and client access to Cloud SQL: 

```shell    
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:$UNICODEX_SA --role roles/run.admin
    
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:$UNICODEX_SA --role roles/cloudsql.client
```

Each `add-iam-policy-binding` command will output information about the policy state, with the output increasing as more bindings are made. 

Grant the automatically generated Cloud Build service account admin access to Cloud Run, and client access Cloud SQL: 

```shell
export PROJECTNUM=$(gcloud projects describe ${PROJECT_ID} --format 'value(projectNumber)')
export CLOUDBUILD_SA=${PROJECTNUM}@cloudbuild.gserviceaccount.com

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member serviceAccount:${CLOUDBUILD_SA} --role roles/cloudsql.client
    
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member serviceAccount:${CLOUDBUILD_SA} --role roles/run.admin
```

Finally, grant the service account permission for Cloud Build to act as our Service Account: 

```shell
gcloud iam service-accounts add-iam-policy-binding ${UNICODEX_SA} \
  --member "serviceAccount:${CLOUDBUILD_SA}" \
  --role "roles/iam.serviceAccountUser"
```


### Create the database

Django supports many databases, but PostgreSQL has the most support. Google Cloud SQL offers managed PostgreSQL database, which can be easily integrated into Django using database connection strings supported by [`django-environ`](https://django-environ.readthedocs.io/).

Create a small Cloud SQL PostgreSQL database: 

```shell
export ROOT_PASSWORD=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 64 | head -n 1)

gcloud sql instances create psql \
  --database-version POSTGRES_11 \
  --tier db-f1-micro  \
  --region $REGION \
  --project $PROJECT_ID \
  --root-password $ROOT_PASSWORD
```

This operation may take a few minutes to complete.

Within your instance, create a database: 

```shell
gcloud sql databases create django-unicodex --instance=psql
```

For your instance, create a database user: 

```shell
export DBPASSWORD=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 40 | head -n 1)

gcloud sql users create unicodex-user \
  --password $DBPASSWORD \
  --instance psql
```

Store a copy of the database connection string using the fully qualified instance name, database name, database user and database password (this will be used later): 

```shell
export DATABASE_URL=postgres://unicodex-user:${DBPASSWORD}@//cloudsql/$PROJECT_ID:$REGION:psql/django-unicodex
```

### Create the storage bucket

Django needs a place to store static and media assets. The sample project uses [`django-storages`](https://django-storages.readthedocs.io) which supports Google Cloud Storage. 

Create a Cloud Storage bucket in your chosen region: 

```shell
export GS_BUCKET_NAME=${PROJECT_ID}-media
gsutil mb -l ${REGION} gs://${GS_BUCKET_NAME}
```

Allow the unicodex service account admin access to the bucket: 

```shell
gsutil iam ch \
  serviceAccount:${UNICODEX_SA}:roles/storage.objectAdmin \
  gs://${GS_BUCKET_NAME}
```

### Store project secrets

The environment variables you have created only exist within your current terminal session. They are best to be stored securely for use by your project. 

Create the `DATABASE_URL` secret: 

```shell
gcloud secrets create DATABASE_URL --replication-policy automatic
echo -n "$DATABASE_URL" | gcloud secrets versions add DATABASE_URL --data-file=-
```

Create the `GS_BUCKET_NAME` secret: 

```shell
gcloud secrets create GS_BUCKET_NAME --replication-policy automatic
echo -n "${GS_BUCKET_NAME}" | gcloud secrets versions add GS_BUCKET_NAME --data-file=-
```


Additionally, create a value for the Django `SECRET_KEY`:

```shell
SECRET_KEY="$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 50 | head -n 1)"

gcloud secrets create SECRET_KEY --replication-policy automatic
echo -n "${SECRET_KEY}" | gcloud secrets versions add SECRET_KEY --data-file=-
```

Grant access to all of these values to the Unicodex service account, and the Cloud Build service account: 

```shell
gcloud secrets add-iam-policy-binding DATABASE_URL \
  --member serviceAccount:${UNICODEX_SA} --role roles/secretmanager.secretAccessor
gcloud secrets add-iam-policy-binding DATABASE_URL \
  --member serviceAccount:${CLOUDBUILD_SA} --role roles/secretmanager.secretAccessor

gcloud secrets add-iam-policy-binding GS_BUCKET_NAME \
  --member serviceAccount:${UNICODEX_SA} --role roles/secretmanager.secretAccessor
gcloud secrets add-iam-policy-binding GS_BUCKET_NAME \
  --member serviceAccount:${CLOUDBUILD_SA} --role roles/secretmanager.secretAccessor

gcloud secrets add-iam-policy-binding SECRET_KEY \
  --member serviceAccount:${UNICODEX_SA} --role roles/secretmanager.secretAccessor
gcloud secrets add-iam-policy-binding SECRET_KEY \
  --member serviceAccount:${CLOUDBUILD_SA} --role roles/secretmanager.secretAccessor
```

### Create and store Django admin secrets

The Django admin requires a superuser to access it. Cloud Run does not have an interactive terminal option, so the sample project uses a [data migration](https://docs.djangoproject.com/en/3.0/topics/migrations/#data-migrations), which will run within Cloud Build. 

Create a superuser name and password, store them as secrets, and grant the Cloud Build service account access: 

```shell
export SUPERUSER="admin"

gcloud secrets create SUPERUSER --replication-policy automatic
echo -n "${SUPERUSER}" | gcloud secrets versions add SUPERUSER --data-file=-
gcloud secrets add-iam-policy-binding SUPERUSER \
    --member serviceAccount:$CLOUDBUILD_SA --role roles/secretmanager.secretAccessor

export SUPERPASS=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 30 | head -n 1)

gcloud secrets create SUPERPASS --replication-policy automatic
echo -n "${SUPERPASS}" | gcloud secrets versions add SUPERPASS --data-file=-
gcloud secrets add-iam-policy-binding SUPERPASS \
    --member serviceAccount:$CLOUDBUILD_SA --role roles/secretmanager.secretAccessor
```

Later in this tutorial you will be logging into the Django admin, and will need to access this password. For now, it is okay to be left unseen. 

## Build, migrate, and deploy

In this section, given the configured backing services, you will manually build, migrate, and deploy the Django project. 


### Copy the sample project code

If you are working in Google Cloud Shell, and previously downloaded the sample code to your local machine, ensure you follow the ["Download the sample application"](#download-the-sample-application) steps to get a copy of the code in your Cloud Shell.

### Build the image

Within the top folder of the sample application code (the top most folder that contains the Dockerfile), build the image: 

```shell
gcloud builds submit --tag gcr.io/$PROJECT_ID/unicodex .
```

### Deploy and configure the service

Using the image just built, deploy the service: 

```shell
gcloud run deploy unicodex \
  --platform managed \
  --region $REGION \
  --allow-unauthenticated \
  --image gcr.io/$PROJECT_ID/unicodex \
  --add-cloudsql-instances $PROJECT_ID:$REGION:psql \
  --service-account $UNICODEX_SA

```

Get the URL that the service was just deployed to: 

```
export SERVICE_URL=$(gcloud run services describe unicodex \
    --format "value(status.url)" --platform managed --region $REGION)
```

Update the service to set the `CURRENT_HOST` value to this URL:   

```shell
gcloud run services update unicodex \
  --platform managed \
  --region $REGION \
  --update-env-vars "CURRENT_HOST=${SERVICE_URL}"
```

ℹ️ This project uses Django's [`ALLOWED_HOSTS`](https://docs.djangoproject.com/en/3.0/ref/settings/#allowed-hosts) functionality, but uses a different environment variable to allow for some manipulation of the passed in values (e.g. removing host schema, setting localhost allowance in some situations)

### Manually run a build, migrate, and deploy

Using the provided Cloud Build configuration, manually run a build, migrate, and deploy: 

```
gcloud builds submit \
  --config .cloudbuild/build-migrate-deploy.yaml \
  --substitutions "_REGION=${REGION},_INSTANCE_NAME=psql,_SERVICE=unicodex"
```

### See your deployed project

Navigate to the `SERVICE_URL` to see your deployed website:

```shell
echo $SERVICE_URL
```

Log into the Django admin using the superuser name and password, retrieving those credentials from Secret Manager: 

```shell
gcloud secrets versions access latest --secret SUPERUSER && echo ""
gcloud secrets versions access latest --secret SUPERPASS && echo ""
```

## Automate deployment

In this section, you will create a copy of the sample application as your own GitHub fork, and automate deployments when to merge code. This step moves away from a static copy of the sample application into a living copy. 

This section is best completed on your local terminal where you have already configured access to GitHub repos. No other programs other than `git` will be use here. 

### Fork and clone the code

Navigate to the [sample django-demo-app-unicodex](https://github.com/GoogleCloudPlatform/django-demo-app-unicodex) project on GitHub and fork the repo. Then, clone the copy of your repo to your local machine: 

```
git clone git@github.com:yourgithubuser/django-demo-app-unicodex
cd django-demo-app-unicodex
git checkout tag/pycon2020 -b master
```

### Connect to the source respository

Before you can create a trigger, you must first [connect your GitHub repo with Cloud Build](https://cloud.google.com/cloud-build/docs/running-builds/create-manage-triggers#connecting_to_source_repositories).

In the Google Cloud Console, go to the [Cloud Build Triggers section](https://console.cloud.google.com/cloud-build/triggers), and click 'Connect Repository'. Select "GitHub (Cloud Build GitHub App)", and follow the authorisation prompts to connect your GitHub account to your Google Cloud account. You will be prompted to install the Google Cloud Build app into your GitHub account; ensure you allow only your `unicodex` fork. 

Back in the Google Cloud interface, select your forked repository, read and confirm the authorisation checkbox note, and click 'Connect repository'. Skip the automatic trigger creation (click 'Skip for now', and click 'Continue'). 


### Creating a trigger

In the main trigger listing, click 'Create Trigger', and make a trigger for your project when you merge to master: 

 * Name: "Master Merge"
 * Event: "Push to branch"
 * Source: 
   * Repository: your fork
   * Branch: "`^master$`"
 * Build configuration
   * Filetype: Cloud Build configuration file
   * Cloud Build configuration file location: `.cloudbuild/build-migrate-deploy.yaml`
 * Substitution variables: 
   * `_REGION`: (the value of `$REGION`)
   * `_INSTANCE_NAME`: psql
   * `_SERVICE`: unicodex


Save this new trigger, and confirm it is listed in the interface. 


Optionally, you can create this trigger using `gcloud`: 

```
gcloud beta builds triggers create github \
  --repo-name django-demo-app-unicodex \
  --repo-owner yourgithubuser \
  --branch-pattern "^master$" \
  --build-config .cloudbuild/build-migrate-deploy.yaml \
  --substitutions "_REGION=${REGION},_INSTANCE_NAME=psql,_SERVICE=unicodex"
```

### Make a change

The currently deployed version of the project has a purple "Unicodex" title. To automate a noticeable change, make a change to this base template. 

As an example: 

 * open the `unicodex/templates/base.html` file, and change the `<h2>` header to say "My Unicodex". 
 * open the `unicodex/static/css/unicodex.css` file, and change the `water` class to a different colour gradient. 



### Trigger the build

Commit your change to your `master` branch, then push your change. 

Since your master branch has been updated, your build will be triggered. Check the status of the build in your [Cloud Build History](https://console.cloud.google.com/cloud-build/builds) (also linked from the orange 'building' dot on the git commit). Once it's successful the orange dot will change to a green checkmark, and your project will have been automatically deployed. 

### Make a more complex change

Since our trigger handles building, migrating, and deploying, it will also handle Django model changes. 

Open the `unicodex/models.py` file, and add a field.  

To create the Django migration, run the `makemigrations` command from Docker Compose to generate the new migration files: 

```
docker-compose run --rm web python manage.py makemigrations
```

ℹ️ Instead of using Docker Compose for this step, you could update the `.cloudbuild/django-migrate.sh` script to run this command for you: 

```shell
echo "🎸 migrate"
python manage.py makemigrations # add this
python manage.py migrate
```

Add your new and changed files to a new commit, and merge those changes. You can optionally create these changes in a branch, create a pull request, and then merge them to master. The deployment will not run until a change to master is made. 


## Automate Provisioning

In this section, you will automate the provisioning steps in the earlier sections of this tutorial by using Terraform manifests to create backing services. 

### Create a new project

In the Google Cloud Console, create a new project for this section. Take note of the new Project ID: 

```
export PROJECT_ID=NewProjectID
gcloud config set project $PROJECT_ID
```

### Setup Terraform

Install [Terraform](https://learn.hashicorp.com/terraform/getting-started/install.html) for your operating system. 

Create a new service account, with editor rights to be able to be able to create all required backing services: 

```
gcloud services enable iam.googleapis.com 
gcloud iam service-accounts create terraform --display-name "Terraform Service Account"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member serviceAccount:terraform@${PROJECT_ID}.iam.gserviceaccount.com \
  --role roles/owner
```

For the newly created service account, create a local private key, and save it to your environment: 

```
gcloud iam service-accounts keys create ~/terraform-key.json \
  --iam-account terraform@${PROJECT_ID}.iam.gserviceaccount.com 

export GOOGLE_APPLICATION_CREDENTIALS=~/terraform-key.json
```

### Build a new copy of the image 

For this newly created project, build a copy of the image: 

```
gcloud services enable cloudbuild.googleapis.com
gcloud builds submit --tag gcr.io/$PROJECT_ID/newunicodex .
```

### Apply Terraform manifests

Navigate to the Terraform manifest directory: 

```
cd terraform
```

Initialise terraform:

```
terraform init
```

Apply the Terraform manifests to your environment, choosing your new project ID, a region, an instance name and a service name: 

```
terraform apply \
  -var "region=us-central1" \
  -var "service=newunicodex" \
  -var "project=$PROJECT_ID" \
  -var "instance_name=newpsql"
```

When prompted, approve the changes by entering "yes". 

⏳ While Terraform is running, in a new terminal or file browser, take a look at the Terraform manifests and how they compare to the original shell commands. 

⚠️ Many Google Cloud components work on an eventual consistency basis. If Terraform fails to complete successfully first time, run the command again until the execution succeeds with "Apply complete! Resources: 0 added, 0 changed, 0 destroyed."

Once terraform has completed, the command required to run the build, migrate, and deploy Cloud Build configuration will be output: 

```
gcloud builds submit --config .cloudbuild/build-migrate-deploy.yaml \
    --substitutions="_REGION=us-central1,_INSTANCE_NAME=newpsql,_SERVICE=newunicodex"
```


Confirm the installation is working by logging into the new service's Django admin using the created superuser name and password, the commands for which were also output:

```
gcloud secrets versions access latest --secret SUPERUSER
gcloud secrets versions access latest --secret SUPERPASS
```



## Cleanup

🧹 Don't forget to clean up!

To avoid incurring charges to your Google Cloud Platform account for the resources used in this tutorial:
 * In the Cloud Console, go to the [Manage resources](https://console.cloud.google.com/cloud-resource-manager) page.
 * In the project list, select your project then click Delete.
 * In the dialog, type the project ID and then click Shut down to delete the project.

