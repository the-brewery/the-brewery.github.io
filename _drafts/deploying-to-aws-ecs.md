---
layout:     page
title:      Automate AWS ECS Deployments
tags:       aws ecs deployment
categories: howtos
author:     Olof Montin
---
Deploy to the AWS ECS can be tricky...

[Snippets](https://bitbucket.org/snippets/thebrewery/9oBR8)

### deploy-to-aws.sh

```bash
#!/bin/bash

# What are we going to deploy?
if [[ "$1" == "dev" ]] || [[ "$1" == "prod" ]]; then
    env="$1"
else
    echo "You'll have to provide an environment: dev or prod"
    exit 2
fi

name="<name>-$env"
image="$name:$SEMAPHORE_BUILD_NUMBER"
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
task=$(aws ecs register-task-definition \
               --family $name --region $region \
               --cli-input-json file://task-definition.json)

if [[ "$task" =~ \"revision\":\ *([0-9]+) ]]; then
    revision=${BASH_REMATCH[1]}
fi

if [[ "$task" =~ \"desiredCount\":\ *([0-9]+) ]]; then
    count=${BASH_REMATCH[1]}
fi

if [[ -z $count ]] || [[ "$count" == "0" ]]; then
    count="1"
fi

# Update the service with the new task definition
aws ecs update-service --cluster default \
        --region $region \
        --service $name \
        --task-definition $name:$revision \
        --desired-count $count
```

### resources/aws-ecs-task-definition.json

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
