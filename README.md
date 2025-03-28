**Deploying to AWS ECS Using GitHub Actions**

This document outlines the step-by-step process to set up Continuous Integration and Continuous Deployment (CI/CD) for deploying a containerized application to Amazon Elastic Container Service (ECS) using GitHub Actions.
Prerequisites

**Before proceeding, ensure you have the following:**

    AWS Account: Access to an AWS account with permissions to create and manage ECS, ECR, IAM, and related resources.

    GitHub Repository: Your application's source code hosted in a GitHub repository.

    Docker: Installed on your local development environment for building container images.

**Steps Performed**
**1. Create an Amazon Elastic Container Registry (ECR)**

Amazon ECR is a fully managed Docker container registry that makes it easy to store, manage, and deploy Docker container images.

    Action: Created a new ECR repository to store the application's Docker images.

    aws ecr create-repository --repository-name your-repo-name --region your-region

    Replace your-repo-name with your desired repository name and your-region with your AWS region.

**2. Build and Push Docker Image to ECR**

    Action: Built the Docker image locally and pushed it to the newly created ECR repository.

    # Authenticate Docker to the ECR registry
    aws ecr get-login-password --region your-region | docker login --username AWS --password-stdin your-account-id.dkr.ecr.your-region.amazonaws.com

    # Build the Docker image
    docker build -t your-repo-name .

    # Tag the Docker image
    docker tag your-repo-name:latest your-account-id.dkr.ecr.your-region.amazonaws.com/your-repo-name:latest

    # Push the Docker image to ECR
    docker push your-account-id.dkr.ecr.your-region.amazonaws.com/your-repo-name:latest

    Ensure to replace placeholders with your actual AWS account ID, repository name, and region.

**3. Create an ECS Cluster**

Amazon ECS clusters are logical groupings of tasks or services.

    Action: Created a new ECS cluster to run the application services.

    aws ecs create-cluster --cluster-name your-cluster-name

**4. Define an ECS Task Definition**

A task definition is a blueprint for your application, specifying containers, CPU, memory, and other configurations.

    Action: Created a task definition JSON file (task-definition.json) with the necessary configurations.

{
  "family": "your-task-family",
  "containerDefinitions": [
    {
      "name": "your-container-name",
      "image": "your-account-id.dkr.ecr.your-region.amazonaws.com/your-repo-name:latest",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80
        }
      ]
    }
  ]
}

Replace placeholders with your actual values.

Action: Registered the task definition with ECS.

    aws ecs register-task-definition --cli-input-json file://task-definition.json

**5. Create an ECS Service**

An ECS service runs and maintains a specified number of instances of a task definition simultaneously in an ECS cluster.

    Action: Created a new ECS service using the registered task definition.

    aws ecs create-service --cluster your-cluster-name --service-name your-service-name --task-definition your-task-family --desired-count 1 --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[subnet-xxxxxxxx],securityGroups=[sg-xxxxxxxx],assignPublicIp=ENABLED}"

    Ensure to specify the correct subnets and security groups.

**6. Set Up GitHub Actions for CI/CD**

GitHub Actions automates workflows directly within GitHub.

    Action: Created a GitHub Actions workflow file (.github/workflows/deploy-to-ecs.yml) to automate the deployment process.

    name: Deploy to AWS ECS

    on:
      push:
        branches:
          - main

    jobs:
      deploy:
        name: Deploy to ECS
        runs-on: ubuntu-latest

        steps:
          - name: Checkout Repository
            uses: actions/checkout@v2

          - name: Configure AWS Credentials
            uses: aws-actions/configure-aws-credentials@v1
            with:
              aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
              aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
              aws-region: your-region

          - name: Login to Amazon ECR
            id: login-ecr
            uses: aws-actions/amazon-ecr-login@v1

          - name: Build, Tag, and Push Docker Image
            env:
              ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
              ECR_REPOSITORY: your-repo-name
              IMAGE_TAG: ${{ github.sha }}
            run: |
              docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
              docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

          - name: Download Existing Task Definition
            run: |
              aws ecs describe-task-definition --task-definition your-task-family --query taskDefinition > task-definition.json

          - name: Update Task Definition with New Image
            run: |
              jq --arg IMAGE "$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" '.containerDefinitions[0].image = $IMAGE' task-definition.json > updated-task-definition.json

          - name: Register New Task Definition
            id: register-task
            run: |
              TASK_DEFINITION_ARN=$(aws ecs register-task-definition --cli-input-json file://updated-task-definition.json | jq -r '.taskDefinition.taskDefinitionArn')
              echo "TASK_DEFINITION_ARN=$TASK_DEFINITION_ARN" >> $GITHUB_ENV

          - name: Update ECS Service with New Task Definition
            run: |
              aws ecs update-service --cluster your-cluster-name --service your-service-name --task-definition $TASK_DEFINITION_ARN

    Ensure to replace placeholders with your actual values.

**7. Configure GitHub Secrets**

To securely store AWS credentials and other sensitive information:

    Action: Added the following secrets to the GitHub repository:

        AWS_ACCESS_KEY_ID: Your AWS access key ID.

        AWS_SECRET_ACCESS_KEY: Your AWS secret access key.

        ECR_REGISTRY: Your ECR registry URI (e.g., your-account-id.dkr.ecr.your-region.amazonaws.com).

# To ensure that your Amazon Elastic Container Service (ECS) tasks automatically pull the latest Docker images from Amazon Elastic Container Registry (ECR), consider the following strategies:​
Repost

# 1. Utilize Unique Image Tags for Each Deployment 

Instead of using a static tag like latest, assign a unique tag to each image version, such as a Git commit hash or a build ID. This practice ensures that ECS recognizes and deploys the new image version.​


# Tag the image with a unique identifier
docker tag your-image:latest your-account-id.dkr.ecr.your-region.amazonaws.com/your-repo-name:your-unique-tag

In your ECS task definition, reference this unique tag:​

"image": "your-account-id.dkr.ecr.your-region.amazonaws.com/your-repo-name:your-unique-tag"

# 2. Automate ECS Service Updates

After pushing a new image to ECR, update your ECS service to use the latest task definition revision. This can be achieved by registering a new task definition with the updated image and forcing a new deployment.​

Steps:

    Register a New Task Definition: Update your task definition with the new image tag and register it.​

    Update the ECS Service: Use the update-service command with the --force-new-deployment flag to initiate a deployment with the new task definition. This forces ECS to pull the latest image from ECR.​

Example:

aws ecs update-service --cluster your-cluster-name --service your-service-name --task-definition your-task-family --force-new-deployment

This approach ensures that your ECS tasks pull the latest image without relying on the latest tag, which can be problematic due to caching behaviors.​

# 3. Leverage AWS CodePipeline for Continuous Deployment

Integrate AWS CodePipeline to automate the build and deployment process. By configuring CodePipeline to detect changes in your source repository, build the Docker image, push it to ECR, and update your ECS service, you achieve a streamlined continuous deployment pipeline.​

Reference: How to automatically deploy new Docker image updates in an ECS service

# 4. Configure ECS Image Pull Behavior

For ECS tasks running on EC2 instances, you can adjust the ECS_IMAGE_PULL_BEHAVIOR parameter to control how images are pulled:​
AWS Documentation

    default: Pulls the image remotely; if the pull fails, uses the cached image.​
    AWS Documentation

    always: Always pulls the image remotely; if the pull fails, the task fails.​
    AWS Documentation

    once: Pulls the image remotely only if it wasn't previously pulled or has been removed from the cache.​

    prefer-cached: Uses the cached image if available; otherwise, pulls it remotely.​

Setting this parameter appropriately can help manage how and when ECS pulls images, balancing between ensuring up-to-date deployments and optimizing startup times.​
AWS Documentation+1Stack Overflow+1

Reference: Container image pull behavior for the EC2 and external launch types for Amazon ECS

# 5. Ensure Proper IAM Permissions

Ensure that your ECS tasks have the necessary IAM permissions to pull images from ECR. For EC2 launch types, this involves attaching the appropriate policies to the instance role. For Fargate launch types, assign the necessary permissions to the task execution role.​
Repost

Reference: Allow Amazon ECS tasks to pull images from Amazon ECR

By implementing these strategies, you can maintain an automated and reliable deployment pipeline, ensuring that your ECS services always run the desired versions of your Docker images.

8. Push Changes and Trigger Deployment

    Action: Committed and pushed changes to the main
