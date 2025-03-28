Deploying to AWS ECS Using GitHub Actions

This document outlines the step-by-step process to set up Continuous Integration and Continuous Deployment (CI/CD) for deploying a containerized application to Amazon Elastic Container Service (ECS) using GitHub Actions.
Prerequisites

Before proceeding, ensure you have the following:

    AWS Account: Access to an AWS account with permissions to create and manage ECS, ECR, IAM, and related resources.

    GitHub Repository: Your application's source code hosted in a GitHub repository.

    Docker: Installed on your local development environment for building container images.

Steps Performed
1. Create an Amazon Elastic Container Registry (ECR)

Amazon ECR is a fully managed Docker container registry that makes it easy to store, manage, and deploy Docker container images.

    Action: Created a new ECR repository to store the application's Docker images.

    aws ecr create-repository --repository-name your-repo-name --region your-region

    Replace your-repo-name with your desired repository name and your-region with your AWS region.

2. Build and Push Docker Image to ECR

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

3. Create an ECS Cluster

Amazon ECS clusters are logical groupings of tasks or services.

    Action: Created a new ECS cluster to run the application services.

    aws ecs create-cluster --cluster-name your-cluster-name

4. Define an ECS Task Definition

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

5. Create an ECS Service

An ECS service runs and maintains a specified number of instances of a task definition simultaneously in an ECS cluster.

    Action: Created a new ECS service using the registered task definition.

    aws ecs create-service --cluster your-cluster-name --service-name your-service-name --task-definition your-task-family --desired-count 1 --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[subnet-xxxxxxxx],securityGroups=[sg-xxxxxxxx],assignPublicIp=ENABLED}"

    Ensure to specify the correct subnets and security groups.

6. Set Up GitHub Actions for CI/CD

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

7. Configure GitHub Secrets

To securely store AWS credentials and other sensitive information:

    Action: Added the following secrets to the GitHub repository:

        AWS_ACCESS_KEY_ID: Your AWS access key ID.

        AWS_SECRET_ACCESS_KEY: Your AWS secret access key.

        ECR_REGISTRY: Your ECR registry URI (e.g., your-account-id.dkr.ecr.your-region.amazonaws.com).

8. Push Changes and Trigger Deployment

    Action: Committed and pushed changes to the main
