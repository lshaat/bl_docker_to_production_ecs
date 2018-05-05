# Builders Lab
## From docker to production on Amazon ECS

### Take aways

By the end of this lab session you'll have seen built a full VPC and Hybrid AWS Fargate and Amazon ECS cluster. Into this cluster you'll learn how to deploy a container service using a CI/CD pipeline.

### Requirements

- AWS account
- AWS CLI installed + configured with your API keys
- Docker engine installed (locally)
- Git CLI installed

### Expected time

expected length of lab

### Solution overview

High level view of the solution we will build on AWS and a list of technologies used.

- Amazon ECS
- Amazon ECR
- AWS CodeCommit
- AWS CodePipeline
- AWS CodeBuild

Diagrams of the solution

## Exercise
### Getting started

Building a docker image: include code for app

```bash
code block with language tag
```

### Runnign locally

Run and test locally using docker

### Starting a cluster

VPC design

Cluster creation

ALB integration

### Version control and docker registry

Commit code to code commit and manually push a build to ECR

### Automating docker builds

This tutorial uses AWS CodeBuild to build your Docker image and push the image to Amazon ECR. Add a buildspec.yml file to your source code repository to tells AWS CodeBuild how to do that. The example build specification below does the following:

- Pre-build stage:
 - Log in to Amazon ECR.
 - Set the repository URI to your ECR image and add an image tag with the first seven characters of the Git commit ID of the source.

- Build stage:
 - Build the Docker image and tag the image both as latest and with the Git commit ID.

- Post-build stage:
 - Push the image to your ECR repository with both tags.
 - Write a file called imagedefinitions.json in the build root that has your Amazon ECS service's container name and the image and tag. The deployment stage of your CD pipeline uses this information to create a new revision of your service's task definition, and then it updates the service to use the new task definition. The imagedefinitions.json file is required for the AWS CodeDeploy ECS job worker.


```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - $(aws ecr get-login --region eu-west-1 --no-include-email)
      - REPOSITORY_URI=210944566071.dkr.ecr.eu-west-1.amazonaws.com/osjs
      - IMAGE_TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '[{"name":"osjs","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
artifacts:
    files: imagedefinitions.json
```

The build specification was written for the following task definition, used by the Amazon ECS service for this tutorial. The REPOSITORY_URI value corresponds to the image repository (without any image tag), and the osjs value near the end of the file corresponds to the container name in the service's task definition.  

TASK DEF HERE

SERVCIE DEF HERE



### Automating deployments

Deploying on new version builds

### What next?

Fargate