# tiny-house-dashboard

## Python Dev Tools Setup
To install the env, run `pipenv install --dev`  
To initialize the git hooks, run `pipenv run pre-commit install`  
If there are file that were committed before adding the git hooks, run `pipenv run pre-commit run --all-files`  
To run tests, run `pipenv run pytest --cov-config=.coveragerc --cov=resc`  

## Start the App Locally
Copy `demo.env` to `.env`: `cp demo.env .env`  
Change `MAPBOX_TOKEN` in `.env` to your Mapbox Token  
Run the app locally in dev mode with `pipenv run python app/app.py`  

## Deploy to GCP

### Skip CGR and Deploy with App Engine
Note: there can only be one app engine application per project.  
Create the App Engine App: `gcloud app create`  
Deploy with `gcloud app deploy --appyaml=app.yml`


### Build and Deploy to GCR
To select you account and project, run `gcloud init`  
Set a default region.  
Set an `APP_NAME` environmental variable with: `export APP_NAME=my-app`
To get and save your project id, run `export PROJECT=$(gcloud config get-value project)`  
To get the GCR tag, run `export GCR_TAG=gcr.io/$PROJECT/$APP_NAME`  
To submit the build to GCP Cloud Build, run `gcloud builds submit --tag $GCR_TAG`  
To see the GCR tag, run `echo $GCR_TAG`  

### Use the Console to Deploy to GCE and Start Container (option 1)
Go to "Compute Engine" in the GCP UI and select "CREATE INSTANCE".  
In the options select the checkbox "Deploy a container image to this VM instance."  
Paste in the GCR tag in the "Container Image" field.  
Click "Advanced container options".  
Add the Mapbox token as an environment variables.  
Select "Allow HTTP traffic" for a public deployment.  


### Use the SDK to Deploy to GCE and Start Container (option 2)

Run:
```
gcloud compute instances create-with-container $APP_NAME \
  --machine-type e2-small \
  --container-image $GCR_TAG \
  --tags dash-server \
  --container-env-file .env

```

Run:
```
gcloud compute firewall-rules create dash-allow-http \
  --allow tcp:80 \
  --target-tags dash-server
``` 

### Use the Console to Deploy to Cloud Run (option 3)
Go to "Cloud Run" in the GCP console and select "CREATE Service".  
Pick a service name and Region.  
Select "Deploy one revision from an existing container image" and select the latest image image for "gcr.io/[your project]/[app name]".  
Select "Allow all traffic".  
Select "Allow unauthenticated invocations".
Click "Create".  
Click "EDIT & DEPLOY NEW REVISION".  
Click "CONTAINER" and change "Container port" to 80.
Click "VARIABLES & SECRETS".
Click "ADD VARIABLE" and add the variable with the Name "MAPBOX_TOKEN" and the Value as the token.  
Click "Deploy".  

### Use the SDK to Deploy to Cloud Run (option 4)

#### Set Deploy Time Environmental Variables

Environmental variables are a good way to keep our tokens secret and our options configurable.  
To set them, copy sample.envrc to .envrc and change the values.  
To load .envrc, one could just run .envrc as a shell script, but direnv will make things easier by automatically loading the variable when you enter the directory, once allowed.  
To install direnv on a mac running zsh, use brew to install with brew install direnv and hook in into your shell by adding eval "$(direnv hook zsh)" to your .zshrc file.  
For other install instructions see: https://direnv.net/.  
To allow direnv to load .envrc in a directory run direnv allow.  

#### Deploy
Run:
```
gcloud run deploy $APP_NAME --image $GCR_TAG \
   --platform managed \
   --allow-unauthenticated \
   --set-env-vars MAPBOX_TOKEN=$MAPBOX_TOKEN \
   --region us-west1 \
   --port 80
```

## Clean Up

### GCR
Every deployment option creates a GCR image. Delete with: `gcloud container images delete gcr.io/$PROJECT/$APP_NAME`

There are some artifacts that are created by GCR. Delete them using `gsutil`.  
If you don't have it installed see: https://cloud.google.com/sdk/docs/install  
NOTE: THIS WILL DELETE OTHER ARTIFACTS AS WELL, RUN WITH CARE.  
Run:
```
gsutil rm -r gs://artifacts.$PROJECT.appspot.com && \
gsutil rm -r gs://${PROJECT}_cloudbuild
```

### Clean Up GCE Manually (option 1)
If you used option 1 to deploy, go to "Compute Engine" in the console and delete the VM that you made.  

### Clean Up GCE Automatically (option 2)
If you used option 2 to deploy, run: `gcloud compute instances delete $APP_NAME`  

### Clean Up Cloud Run Manually (option 3)
If you used option 3 to deploy, go to "Cloud Run" in the console and delete the service that you made.  

### Clean Up Cloud Run Automatically (option 4)
If you used option 4 to deploy, run `gcloud run services delete $APP_NAME --region us-west1`  
