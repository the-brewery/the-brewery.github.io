---
layout:     page
title:      Automate AWS ECS Deployments
tags:       aws ecs deployment
categories: howtos
author:     Olof Montin
---
[AWS ECS](https://aws.amazon.com/ecs/) is a [docker container](https://docker.com/) service from [Amazon Web Services](https://aws.amazon.com/). This article is not about docker and how to containerize your application. This article is about how to deploy containers in some already defined ECS cluster setup at AWS.

So, I guess you already been setting up your cluster at [AWS ECS](https://aws.amazon.com/ecs/) with some mixed results. Usually I'm setting up our clusters by defining tasks and services, one by one. Then adding load balancing and auto scaling. And not using the guide, which usually fails.

Then, when your services are running within the [AWS ECS](https://aws.amazon.com/ecs/), you'll probably want to automate your deployment. There are several ways if doing this, but this is the way I thought was the best:

1. Hook up your repository with some continuos integration that builds your docker image and test it.
2. When all tests are passing, tag your image with a name and a build-number or version and push it to AWS ECS's docker registry.

   ```bash
   docker tag '<image-name' '<registry-id>.dkr.ecr.<region>.amazonaws.com/<image-name>'
   docker push '<registry-id>.dkr.ecr.<region>.amazonaws.com/<image-name>'
   ```

3. Define a new task definition with your image.

   ```bash
  aws ecs register-task-definition \
          --family '<task-name>' --region '<region>' \
          --cli-input-json file://task-definition.json
   ```

4. And then update the service to run the latest task.

   ```bash
  aws ecs update-service --cluster '<cluster-name>' \
          --region '<region>' \
          --service '<service-name>' \
          --task-definition '<task-name>':'<task-revision>' \
          --desired-count '<desired-count-of-running-containers>'
   ```

In this way:

* You can easely track which image revisions that runs and do rollbacks if needed.
* And get seamless deploys without downtime as the service will start up new tasks and slowly shutdown the previous revisions as the requests starts to fade out.

For this a wrote a small bash script that automates this. And a task definition template. You'll find them below with inline comments. Please use and rewrite at your own risk.

### deploy-to-aws.sh

The deploy script

```bash
#!/bin/bash

##
# This script builds and deploys a Node.js docker image to AWS ECS

# First, require an environment
# What are we going to deploy?
if [[ "$1" == "dev" ]] || [[ "$1" == "prod" ]]; then
    env="$1"
else
    echo "You'll have to provide an environment: dev or prod"
    exit 2
fi

# Then define variables for image, region and registry
name="<name>-$env"
image="$name:$SOME_BUILD_NUMBER_FROM_YOUR_CI"
region="<region>"
registry_id="<registry-id>"
registry="$registry_id.dkr.ecr.$region.amazonaws.com/$image"

# Login to docker service through aws
`aws ecr get-login --region $region`

# Build dependencies
npm install

# Build docker image
docker build -t $name .

# Tag docker image
docker tag $name $registry

# Push image to aws docker registry
docker push $registry

# Use the task definition json in resources as a template
# for generating a new task definition for this build
sed -e "s/__IMAGE_NAME__/$image/g" \
    resources/aws-ecs-task-definition.json | \
    sed -e "s/__ENV__/$env/g" | \
    sed -e "s/__CONTAINER_NAME__/$name/g" > task-definition.json

# Create a new task definition
# and save the output json to use when updating the service
task=$(aws ecs register-task-definition \
               --family $name --region $region \
               --cli-input-json file://task-definition.json)

# Fetch the task revision
if [[ "$task" =~ \"revision\":\ *([0-9]+) ]]; then
    revision=${BASH_REMATCH[1]}
fi

# Fetch the desired count of running containers
if [[ "$task" =~ \"desiredCount\":\ *([0-9]+) ]]; then
    count=${BASH_REMATCH[1]}
fi

# Fallback to one container of we didn't find any desired count
if [[ -z $count ]] || [[ "$count" == "0" ]]; then
    count="1"
fi

# Update the service with the new task definition
# which starts up the new tasks and slowly shutdown the previous
aws ecs update-service --cluster default \
        --region $region \
        --service $name \
        --task-definition $name:$revision \
        --desired-count $count
```

### resources/aws-ecs-task-definition.json

The task definition template

```bash
{
  "networkMode": "bridge",
  "taskRoleArn": "arn:aws:iam::<registry-id>:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "memory": 300,
      "portMappings": [
        {
          "hostPort": 0,
          "containerPort": 5000,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "name": "__CONTAINER_NAME__",
      "environment": [{
        "name": "NODE_ENV",
        "value": "__ENV__"
      }],
      "image": "<registry-id>.dkr.ecr.<region>.amazonaws.com/__IMAGE_NAME__",
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "<awslog-group-name>",
          "awslogs-region": "<region>"
        }
      },
      "cpu": 10
    }
  ],
  "family": "__IMAGE_NAME__"
}
```

Scripts are also available at [Bitbucket](https://bitbucket.org/snippets/thebrewery/9oBR8)
