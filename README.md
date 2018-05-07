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

We are going to use the AWS technologies listed below to build a CI/CD pipeline that once you commit changes to your master branch of your docker repository in git a rebuild and redeploy process will kick off, allowing you to update and test teh docker image locally and then deploy to production easily.

Products used:

- Amazon ECS
- Amazon ECR
- AWS CodeCommit
- AWS CodePipeline
- AWS CodeBuild

Diagrams of the solution

## Exercise
### Getting started

First we need to build an image and test it. For this we are using the [OS.js](https://www.os-js.org/) project which is a virtual desktop you can run in your browser. First lets prepare and build your image.

For this we are going to create a new AWS CodeCommit repository. But lets make sure you have ssh access to your git repositories first.

Generate a new SSH key:

```bash
ssh-keygen

Generating public/private rsa key pair.
Enter file in which to save the key (/drive/Users/user-name/.ssh/id_rsa): Type a file name here, for example /c/Users/user-name/.ssh/codecommit_rsa

Enter passphrase (empty for no passphrase): <Type a passphrase, and then press Enter>
Enter same passphrase again: <Type the passphrase again, and then press Enter>

Your identification has been saved in drive/Users/user-name/.ssh/codecommit_rsa.
Your public key has been saved in drive/Users/user-name/.ssh/codecommit_rsa.pub.
The key fingerprint is:
45:63:d5:99:0e:99:73:50:5e:d4:b3:2d:86:4a:2c:14 user-name@client-name
The key's randomart image is:
+--[ RSA 2048]----+
|        E.+.o*.++|
|        .o .=.=o.|
|       . ..  *. +|
|        ..o . +..|
|        So . . . |
|          .      |
|                 |
|                 |
|                 |
+-----------------+
```
This generates:

- The codecommit_rsa file, which is the private key file.
- The codecommit_rsa.pub file, which is the public key file.

Copy the contents of codecommit_rsa.pub (it may be in your .ssh directoty)

Now Sign in to [https://console.aws.amazon.com/iam/](https://console.aws.amazon.com/iam/)

In the IAM console, in the navigation pane, choose Users, and from the list of users, choose your IAM user.

On the user details page, choose the Security Credentials tab, and then choose Upload SSH public key.

Paste the contents of your SSH public key into the field, and then choose Upload SSH public key.

Copy or save the information in SSH Key ID (for example, APKAEIBAERJR2EXAMPLE). This is your SSH username.

Create a file called ~/.ssh/config and add the following information taking note to change User and IdentityFile Locations

```
Host git-codecommit.*.amazonaws.com
  User APKAEIBAERJR2EXAMPLE
  IdentityFile ~/.ssh/codecommit_rsa
```

Now lets create and clone a repository:

```bash
aws codecommit create-repository --repository-name osjs --region eu-west-1
{
    "repositoryMetadata": {
        "accountId": "210944566071",
        "repositoryId": "271c3e97-6a6a-4e4c-a3d7-de7f0e272897",
        "repositoryName": "osjs",
        "lastModifiedDate": 1525696265.885,
        "creationDate": 1525696265.885,
        "cloneUrlHttp": "https://git-codecommit.eu-west-1.amazonaws.com/v1/repos/osjs",
        "cloneUrlSsh": "ssh://git-codecommit.eu-west-1.amazonaws.com/v1/repos/osjs",
        "Arn": "arn:aws:codecommit:eu-west-1:210944566071:osjs"
    }
}
```
Copy the cloneUrlSsh value and run:

```bash
git clone ssh://git-codecommit.eu-west-1.amazonaws.com/v1/repos/osjs
```

### Runnign locally

Now lets create our docker image and test locally in the root of the osjs directory create a new file called DockerFile and add the following conents:

```docker
FROM debian:latest
MAINTAINER Ric Harvey <rjh@amazon.com>

RUN apt-get update && \
    apt-get install -y curl gnupg git && \
    curl -sL https://deb.nodesource.com/setup_8.x | bash - && \
    apt-get install -y nodejs

RUN git clone https://github.com/os-js/OS.js.git && cd OS.js

WORKDIR OS.js

RUN npm install
RUN node osjs build

CMD node osjs run
```

This is going to build the OSJS software on a debian base image.

**Note:** you should consider stripping down this file to get a smaller build.

Lets build and test the image:

```bash
docker build -t osjs:latest .
docker run -p 8000:8000 osjs:latest
```

Now head to your web browser and go to http://localhost:8000 you should get something like the following:

![OSJS Desktop](./img/desktop.png)

Have a play and take mental note of the menu options. This is how we'll test CI/CD is working later.

### Starting a cluster

Ok so we now have a working docker image locally that we can debug and test with. But what about deploying this in production? Lets create an ECS cluster in our AWS account ready to take the containers.

##### VPC design

First we are going to need a VPC into which we'll deploy our ECS instances. We are going to use cloudformation and deploy three public subnets and three private subnets, our ECS instances will be in the provate subnet and we'll use load balancers to make this publically accessible.

In this git repository there is a folder called cloudformation. This contains our files for building our VPC.

Open the AWS console in your browser and go to cloudformation. Press Create Stack.

![New Stack](./img/newstack.png) 

Select upload template to S3 and find the file named **_public-private-vpc.yml_**

![Template](./img/template.png)

Answer a few questions such as stack name and set your Environment variable. This varibale needs to be the same in the next twoo templates.

Wait for this to complete in the console.

##### Cluster creation

Once its completed we are going to upload two more stacks. One to create a ECS cluster namespace that can run fargate or ECS containers, we'll also create an internal and external facing ALB for connecting to our containers. The second file will bring three ECS hosts to your ECS cluster config, one in each AZ.

Press Create Stack again and upload the **_cluster.yml_** file. Once complete head over to the ECS console and copy the cluster name of your newly created cluster.

Now go back to Cloudformation and create the final stack in the same way. this one is called **_ecs-hosts.yml_**. You'll need to enter the environment name and the cluster name you just copied. once the hosts start you have a fully working cluster.

When creating this document I created an environment called 'dev' I ended up with three stacks created.

![Stacks](./img/stacks.png)

### Version control and docker registry

Right we need to do two things now. Firstly lets commit our code to git. Secondly we'll create a docker repository for the OSJS image we built locally. In your terminal go to the git repository you cloned earlier and run the following commands:

```bash
git add .
git commit -am "first commit"
git push
```

Now go to the ECS console in your web browser and click on repositories.


Create a new repository called osjs:

![Create a New Repo](./img/createrepo.png)

Run the following command in your terminal:

```bash
$(aws ecr get-login --no-include-email --region eu-west-1)
```

This will set up docker with credentials to be able to use the docker tools fromt he commandline.

Now lets tag and push the image we built earlier:

```bash
docker tag osjs:latest 210944566071.dkr.ecr.eu-west-1.amazonaws.com/osjs:latest

docker push 210944566071.dkr.ecr.eu-west-1.amazonaws.com/osjs:latest
```

Your image is now uploaded and ready to be used.

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