# Onward Logistics order system: infrastructure

AWS infrastructure for the Onward Logistics order system on Amazon ECS Fargate, and the pipeline that releases new versions with no downtime.

**Architecture diagram:** [open in draw.io](https://app.diagrams.net/#Uhttps%3A%2F%2Fraw.githubusercontent.com%2Feffienjoki%2Fonward-orders-infrastructure%2Fmain%2Fdocs%2Farchitecture%2Fonward-lab2-architecture.drawio)

**Discovery report:** [docs/discovery-report.md](docs/discovery-report.md)

## The three repositories

| Repository | Job | Deployed by |
| --- | --- | --- |
| [onward-orders-bootstrap](https://github.com/effienjoki/onward-orders-bootstrap) | What must exist first: ECR repository, GitHub OIDC provider, GitHub Actions role | CloudFormation Git sync |
| [onward-orders-app](https://github.com/effienjoki/onward-orders-app) | The Java app, Dockerfile, build workflow and release files | GitHub Actions |
| **onward-orders-infrastructure** (this repo) | Network, ECS service and delivery pipeline | CloudFormation Git sync |

## What is in this repo

| Template | Stack | Creates |
| --- | --- | --- |
| `templates/network.yaml` | `onward-dev-ecs-network` | VPC across two Availability Zones, public subnets for the load balancer, private subnets for tasks with no internet route, VPC endpoints for ECR, CloudWatch Logs and S3 |
| `templates/service.yaml` | `onward-dev-ecs-service` | Internet facing ALB, blue and green target groups, ECS cluster, task definition, ECS service (2 to 6 tasks, scaling on CPU), log group |
| `templates/delivery.yaml` | `onward-dev-ecs-delivery` | EventBridge rule, CodePipeline, CodeDeploy blue/green deployment group, artifact bucket |

Each template has a matching file in `deployments/` that tells Git sync which template to deploy, with which parameters and tags.

## How the stacks connect

Stacks share values with exports and imports:

- **network** exports the VPC and subnet IDs, used by **service**
- **service** exports the cluster, service, listener, target groups and execution role, used by **delivery**
- **bootstrap** (separate repo) exports the ECR repository, used by **service** and **delivery**

## How a release works

1. An engineer merges a change to `main` in `onward-orders-app`. This is the only manual step.
2. GitHub Actions builds the image, gets temporary AWS credentials through OIDC, and pushes it to ECR as `:<commit>` and `:latest`.
3. EventBridge sees the new `:latest` image and starts CodePipeline.
4. CodePipeline reads `deploy/taskdef.json` and `deploy/appspec.yaml` from the app repo and hands them, with the exact image, to CodeDeploy.
5. CodeDeploy starts green tasks, sends 10 percent of traffic to them for 5 minutes, then 100 percent. Blue tasks are kept for 10 minutes so the release can be reversed instantly.

## How to roll back

- **During a release, or within 10 minutes after:** CodeDeploy console, open the deployment, choose **Stop deployment** and **Stop and roll back**. Traffic returns to blue.
- **Later:** revert the commit in `onward-orders-app` and merge. The previous version is released through the same pipeline.

## Security

- No long lived credentials anywhere. GitHub uses OIDC. AWS services use IAM roles.
- Only the load balancer is public. Tasks run in private subnets with no route to the internet and reach AWS only through VPC endpoints.
- Tasks accept traffic only from the load balancer's security group. No admin ports.
- No inline IAM policies. Roles use AWS managed or customer managed policies.

## Rebuild from scratch

Deploy in this order. Each step depends on the one before.

1. `onward-orders-bootstrap` (Git sync)
2. Push to `main` in `onward-orders-app`, so ECR has a first image
3. `onward-dev-ecs-network`, deployment file `deployments/network.yaml`
4. `onward-dev-ecs-service`, deployment file `deployments/service.yaml`
5. `onward-dev-ecs-delivery`, deployment file `deployments/delivery.yaml`

**Delete in reverse order:** delivery, service, network, then bootstrap. Empty the artifact bucket before deleting the delivery stack.

## Prerequisites created once (bootstrap exceptions)

- GitHub connection used by Git sync and CodePipeline
- CloudFormation deploy role `onward-dev-cloudformation-deploy-role`
- One Git sync role per stack, created when the stack is linked

## Standards

Names follow `onward-<env>-<purpose>`. Every stack is tagged `Project=onward`, `Owner=Effie Njoki`, `Environment=dev`. Region: `eu-west-1`.