# bitbucket-jenkins-acs-pipeline

Ref: http://abakonski.com/

An example Jenkins pipeline, triggered by BitBucket pushes, deploying to Azure Container Service.
Uses Azure Container Registry as a Docker image registry.
Includes Slack integration to communicate deployment progress

The sample app intended to be deployed in this example can be found here: https://github.com/abakonski/docker-compose-flask