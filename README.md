# ☁️ MultiCloud, DevOps & AI Challenge - Day 3

This repository contains the implementation for Day 3 of the MultiCloud, DevOps & AI Challenge, focusing CI/CD Pipeline Configuration CloudMart application.

## Overview

The Cloud App will be put on autopilot with DevOps CI/CD pipelines on AWS CodePipeline and deploy the source code changes

## Architecture
Amazon Web Services
Github Codespaces
AWS CodePipeline
AWS CodeBuild
Docker
Amazon Elastic Container Registry

## Key Features
✅Automated CI/CD Pipeline:

Built a fully automated CI/CD pipeline using AWS CodePipeline to streamline the testing and deployment process for an E-commerce application.
Every push to the GitHub repository triggers the pipeline, ensuring seamless and continuous delivery.

✅Integration with GitHub:
Connected the pipeline to a GitHub repository to monitor changes in the main branch.
Used GitHub OAuth tokens for secure integration with AWS CodePipeline.

✅Automated Builds with AWS CodeBuild:
Leveraged AWS CodeBuild to automatically build the application whenever changes are pushed to the repository.
Configured build specifications (buildspec.yml) to define build steps, such as installing dependencies, running tests, and packaging the application.


## Part 1: CI/CD Pipeline Configuration

##  Configure AWS CodePipeline
1. **Create a New Pipeline:**
    - Access AWS CodePipeline.
    - Start the 'Create pipeline' process.
    - Name: `cloudmart-cicd-pipeline`
    - Use the GitHub repository `cloudmart-application` as the source.
    - Add the 'cloudmartBuild' project as the build stage.
    - Add the 'cloudmartDeploy' project as the deployment stage.
      
### Step 1: Configure AWS CodeBuild to Build the Docker Image

- Clone Frontend on your Repo
```bash
cd challenge-day2/frontend
<Run GitHub steps to clone>
```
  1. **Create a Build Project:**
    - Give the project a name (for example, **`cloudmartBuild`**).
    - Connect it to your existing GitHub repository (**`cloudmart-application`**).
    - **Image: amazonlinux2-x86_64-standard:4.0**
    - Configure the environment to support Docker builds. Enable "Enable this flag if you want to build Docker images or want your builds to get elevated privileges"
    - Add the environment variable **ECR_REPO** with the ECR repository URI.
    - For the build specification, use the following **`buildspec.yml`**:
     
```bash
version: 0.2
phases:
  install:
    runtime-versions:
      docker: 20
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - REPOSITORY_URI=$ECR_REPO
      - aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/l4c0j8h9
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - export imageTag=$CODEBUILD_RESOLVED_SOURCE_VERSION
      - printf '[{\"name\":\"cloudmart-app\",\"imageUri\":\"%s\"}]' $REPOSITORY_URI:$imageTag > imagedefinitions.json
      - cat imagedefinitions.json
      - ls -l

env:
  exported-variables: ["imageTag"]

artifacts:
  files:
    - imagedefinitions.json
    - cloudmart-frontend.yaml

```

- Add the AmazonElasticContainerRegistryPublicFullAccess permission to ECR in the service role
Access the IAM console > Roles.
Look for the role created "cloudmartBuild" for CodeBuild.
Add the permission **AmazonElasticContainerRegistryPublicFullAccess**.

![Capture d’écran 2025-02-28 011318](https://github.com/user-attachments/assets/7a65db0d-ea4c-48b0-9071-74c2e606d74a)


  ### Step 2: Configure AWS CodeBuild for Application Deployment
  
Repeat the process of creating projects in CodeBuild.
Give this project a different name (for example, **`cloudmartDeployToProduction`**).
Configure the environment variables AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY for the credentials of the user **`eks-user`** in Cloud Build, so it can authenticate to the Kubernetes cluster.
For the deployment specification, use the following buildspec.yml:

```bash
version: 0.2

phases:
  install:
    runtime-versions:
      docker: 20
    commands:
      - curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl
      - chmod +x ./kubectl
      - mv ./kubectl /usr/local/bin
      - kubectl version --short --client
  post_build:
    commands:
      - aws eks update-kubeconfig --region us-east-1 --name cloudmart
      - kubectl get nodes
      - ls
      - IMAGE_URI=$(jq -r '.[0].imageUri' imagedefinitions.json)
      - echo $IMAGE_URI
      - sed -i "s|CONTAINER_IMAGE|$IMAGE_URI|g" cloudmart-frontend.yaml
      - kubectl apply -f cloudmart-frontend.yaml


```
Replace the image URI on line 18 of the cloudmart-frontend.yaml files with CONTAINER_IMAGE.

```bash
git add -A
git commit -m "replaced image uri with CONTAINER_IMAGE"
git push
```

## Part 2: Test your CI/CD Pipeline
- Make a Change on GitHub:
Update the application code in the cloudmart-application repository on the File src/components/MainPage/index.jsx line 93.
![Capture d’écran 2025-02-28 094407](https://github.com/user-attachments/assets/944da0c9-7070-4b82-8c86-888c310b89c9)
![Capture d’écran 2025-02-28 094538](https://github.com/user-attachments/assets/e7bfeb1b-75ca-42d6-a157-d6e3c365efeb)

Commit and push the changes

``` bash
git add -A
git commit -m "changed to Featured Products on CloudMart"
git push
```
# BEFORE

![Capture d’écran 2025-02-27 010752](https://github.com/user-attachments/assets/58eb52ed-30af-444d-889a-b808321cc64f)

# AFTER

![Capture d’écran 2025-02-28 103305](https://github.com/user-attachments/assets/ba8e7e06-a912-43e4-9486-11b2504ddf9f)

# BUILD CONTENT
![Capture d’écran 2025-02-28 103305](https://github.com/user-attachments/assets/ba64daa5-bebf-4312-8f42-fbd1fc6702f5)

# DEPLOY CONTENT
![Capture d’écran 2025-02-28 103455](https://github.com/user-attachments/assets/72478cf2-f3d6-4dd7-9fdf-7168ceeebc10)
