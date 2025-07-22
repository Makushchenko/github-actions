# GitHub Actions: Deploy to Amazon ECS

This workflow automates building and pushing a Docker image to Amazon ECR and deploying a new task definition to Amazon ECS whenever code is pushed to the `master` branch.

## Workflow File

Place this file in your repository under `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Amazon ECS

on:
  push:
    branches:
      - "master"

env:
  AWS_REGION:             "us-east-1"
  ECR_REPOSITORY:         "nginx-image-storage"
  ECS_SERVICE:            "Custom-server-service"
  ECS_CLUSTER:            "Custom-server-cluster"
  ECS_TASK_DEFINITION:    "task-definition-template.json"
  CONTAINER_NAME:         "Custom-server-container"
  AWS_EXECUTION_TASK_ROLE: "arn:aws:iam::111111111111:role/ecsTaskExecutionRole"

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region:            ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG:    latest  # or ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push  $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "::set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

      - name: Render new ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: ${{ env.ECS_TASK_DEFINITION }}
          container-name:  ${{ env.CONTAINER_NAME }}
          image:           ${{ steps.build-image.outputs.image }}

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition:           ${{ steps.task-def.outputs.task-definition }}
          service:                   ${{ env.ECS_SERVICE }}
          cluster:                   ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```

## Prerequisites

1. **ECR Repository**: Create or use an existing ECR repo.

   ```bash
   aws ecr create-repository --repository-name nginx-image-storage --region us-east-1
   ```
2. **ECS Cluster & Service**: Provision your ECS cluster, task definition, and service.
3. **IAM Permissions**: Store `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in GitHub secrets. Ensure the IAM user has:

   * `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`
   * `ecs:RegisterTaskDefinition`, `ecs:UpdateService`, `ecs:DescribeServices`, `ecs:DescribeTaskDefinition`
   * `iam:PassRole` (for your execution role)

## Environment Variables

| Variable                  | Description                                       |
| ------------------------- | ------------------------------------------------- |
| `AWS_REGION`              | AWS region for ECR & ECS (e.g. `us-east-1`).      |
| `ECR_REPOSITORY`          | ECR repository name (e.g. `nginx-image-storage`). |
| `ECS_CLUSTER`             | ECS cluster name.                                 |
| `ECS_SERVICE`             | ECS service name.                                 |
| `ECS_TASK_DEFINITION`     | Path to your task definition JSON in the repo.    |
| `CONTAINER_NAME`          | The container name in your task definition.       |
| `AWS_EXECUTION_TASK_ROLE` | ARN of ECS task execution IAM role.               |

## Usage

1. Push changes to `master`.
2. Workflow builds and pushes your Docker image to ECR.
3. ECS service is updated with the new task definition.

---

*Ensure your JSON task definition file is committed and that the container name matches exactly.*