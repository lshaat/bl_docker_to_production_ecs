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
- AWS Code Commit
- AWS Code Pipeline
- AWS Code Build

Diagrams of the solution

## Exercise
### Getting started

Building a docker image: include code for app

```bash
code block with language tag
```

### Runnign locally

Run and test locally using docker

### Version control and docker registry

Commit code to code commit and manually push a build to ECR

### Automating docker builds

Use code pipeline na dcode build to generate images and push to ECS

### Starting a cluster

VPC design

Cluster creation

ALB integration

### Automating deployments

Deploying on new version builds

### What next?

Fargate