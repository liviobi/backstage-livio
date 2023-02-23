# Backstage Bip Setup

## Local Dev Deployment

To deploy dev environment locally please refer to the the docs ([Backstage - Getting Started](https://backstage.io/docs/getting-started/))

## Docker Compose deployment

1. Follow the the steps to create a backstage app as presented in the docs ([Backstage - Getting Started](https://backstage.io/docs/getting-started/))
2. Build the Docker image as presented below (or see the [docs](https://backstage.io/docs/deployment/docker))

   - Run these commands in the root of the project folder

     ```sh
     yarn install
     yarn tsc
     yarn build:backend
     ```

   - Build the docker image

     ```sh
     docker image build . -f packages/backend/Dockerfile -t backstage
     ```

3. Pull Postgres image
   ```sh
   docker pull postgres
   ```
4. Create `docker-compose.yml` file in the root of the project

   ```yml
   version: '3'

   services:
     postgres:
       image: postgres
       environment:
         POSTGRES_USER: postgres
         POSTGRES_PASSWORD: postgres
         POSTGRES_DB: backstage
       ports:
         - '5432:5432'
       volumes:
         - postgres-data:/var/lib/postgresql/data

     backstage:
       image: backstage
       environment:
         BACKSTAGE_APP_URL: 'http://localhost:7007'
         BACKSTAGE_APP_TITLE: 'Backstage Docker'
         BACKSTAGE_APP_ORG_NAME: 'Bip Org'
         BACKSTAGE_APP_CORS: '0.0.0.0'
         POSTGRES_USER: postgres
         POSTGRES_PASSWORD: postgres
         POSTGRES_DB: backstage
         POSTGRES_HOST: postgres
       ports:
         - '7007:7007'
       depends_on:
         - postgres

   volumes:
     postgres-data:
   ```

   Notice that the configurations that are not set in the `app-config.production.yaml` are take from the `app-config.yaml`

5. Run
   ```sh
   docker-compose up
   ```

## Cloud deployment on Cloud Run

1. Follow the the steps to create a backstage app as presented in the docs ([Backstage - Getting Started](https://backstage.io/docs/getting-started/)) and add it to a [Cloud Build supported version control system](https://cloud.google.com/build/docs/repositories) repository
2. Create a `cloudbuild.yaml` file in the root of the repository:

   ```yaml
    steps:
        - name: 'node:16-bullseye-slim'
        entrypoint: 'yarn'
        args: ['install', '--network-timeout', '300000']
        # Yarn tsc
        - name: 'node:16-bullseye-slim'
          entrypoint: 'yarn'
          args: ['tsc']
        # Yarn build
        - name: 'node:16-bullseye-slim'
          entrypoint: 'yarn'
          args: ['build:backend']
        # Docker Build
        - name: 'gcr.io/cloud-builders/docker'
          args:
            [
              'image',
              'build',
              '.',
              '-f',
              'packages/backend/Dockerfile',
              '-t',
              'gcr.io/${PROJECT_ID}/backstage:xtech-micro-${COMMIT_SHA}',
              '-t',
              'gcr.io/${PROJECT_ID}/backstage:xtech-micro',
            ]
          env:
            - 'DOCKER_BUILDKIT=1'

        # Docker Push
        - name: 'gcr.io/cloud-builders/docker'
          args: ['push', 'gcr.io/${PROJECT_ID}/backstage:xtech-micro-${COMMIT_SHA}']
        # Docker Push
        - name: 'gcr.io/cloud-builders/docker'
          args: ['push', 'gcr.io/${PROJECT_ID}/backstage:xtech-micro']
        # Deploy on cloud Run
        - name: gcr.io/google.com/cloudsdktool/cloud-sdk
          args:
            - run
            - deploy
            - backstage-xtech-micro
            - '--image'
            - 'gcr.io/${PROJECT_ID}/backstage:xtech-micro-${COMMIT_SHA}'
            - '--region'
            - europe-west1
            - '--service-account'
            - '${_SERVICE_ACCOUNT}'
            - >-
              --set-env-vars=POSTGRES_HOST=${_POSTGRES_HOST},POSTGRES_DB=${_POSTGRES_DB},BACKSTAGE_APP_URL=${_BACKSTAGE_APP_URL},BACKSTAGE_APP_TITLE=${_BACKSTAGE_APP_TITLE},BACKSTAGE_APP_CORS=${_BACKSTAGE_APP_CORS},BACKSTAGE_APP_ORG_NAME=${_BACKSTAGE_APP_ORG_NAME}
            - >-
              --set-secrets=POSTGRES_USER=BACKSTAGE_PG_USERNAME:latest,POSTGRES_PASSWORD=BACKSTAGE_PG_PASSWORD:latest
            - '--port'
            - '7007'
            - '--vpc-connector'
            - cloud-native-vpc-connecto
            - '--allow-unauthenticated'
        entrypoint: gcloud
   ```

   ### Cloud Run Deployment Breakdown

   The Cloud Run deployment is done from an image with **gcloud CLI** installed, where we run the command:

   ```sh
       gcloud run deploy
   ```

   and pass all the configurations parameters:
   Parmeter | Description |
   |----------|--------------------------------------------------------------------------------|
   | --image | The image to be deployed on Cloud Run, built and pushed to the registry in the previous steps|
   | --service-account| The service account that runs the job |
   | --set-env-vars | The environment variables passed to the container |  
    | --set-secrets | The secret passed to the container, namely the credentials to access the database |
   | --port | The port that will be open in the container |
   | --vpc-connector | The connector used to connect to the database's vpc |
   | --allow-unauthenticated | Allow access to backstage from everywhere, by default the access is restricted only to authorized users |

3. Connect the repository to Cloud Build
4. Create a trigger with the added repository as source and add the required environment variables:

   | Variable                 | Description                                                                                                                      |
   | ------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
   | \_BACKSTAGE_APP_CORS     | The ip address authorized to make requests to the server, put 0.0.0.0 to allow everyone                                          |
   | \_BACKSTAGE_APP_ORG_NAME | The name of the organization                                                                                                     |
   | \_BACKSTAGE_APP_TITLE    | The title of the webpage                                                                                                         |
   | \_BACKSTAGE_APP_URL      | The URL where the app will be listening inside the container, set it to `http://localhost:7007`                                  |
   | \_POSTGRES_DB            | The name of the Postgres database                                                                                                |
   | \_POSTGRES_HOST          | The private ip of the database, mind that a vpc connection must be enstablished from Cloud Run to the vpc where the db is hosted |

5. When the trigger is run, the docker image of backstage will be built, pushed to container registry and deployed on Google Cloud Run
