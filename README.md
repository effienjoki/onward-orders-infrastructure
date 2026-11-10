# Onward Logistics order system: infrastructure

AWS infrastructure for the Onward Logistics order system on Amazon ECS Fargate, and the pipeline that releases new versions with no downtime. Deployed as one root stack with nested child stacks, by GitHub Actions.

**Architecture diagram:** [open in draw.io](https://app.diagrams.net/#Uhttps%3A%2F%2Fraw.githubusercontent.com%2Feffienjoki%2Fonward-orders-infrastructure%2Fmain%2Fdocs%2Farchitecture%2Fonward-lab2-architecture.drawio)

**Discovery report:** [docs/discovery-report.md](docs/discovery-report.md)

## The three repositories

| Repository | Job | Deployed by |
| --- | --- | --- |
| [onward-orders-bootstrap](https://github.com/effienjoki/onward-orders-bootstrap) | What must exist first: ECR repository, GitHub OIDC provider, both GitHub Actions roles, templates bucket | CloudFormation Git sync (one template) |
| [onward-orders-app](https://github.com/effienjoki/onward-orders-app) | The Java app, Dockerfile, build workflow and release files | GitHub Actions |
| **onward-orders-infrastructure** (this repo) | Network, ECS service and delivery pipeline | GitHub Actions (this repo's workflow) |

## What is in this repo

| Path | Purpose |
| --- | --- |
| `templates/root.yaml` | Root stack. Creates the three child stacks and passes values between them |
| `templates/network.yaml` | Child: VPC across two Availability Zones, public subnets for the load balancer, private subnets for tasks with no internet route, VPC endpoints for ECR, CloudWatch Logs and S3 |
| `templates/service.yaml` | Child: internet facing ALB, blue and green target groups, ECS cluster, task definition, ECS service, CPU scaling, log group |
| `templates/delivery.yaml` | Child: EventBridge rule, CodePipeline, CodeDeploy blue/green deployment group, artifact bucket |
| `.github/workflows/deploy-infrastructure.yml` | Validates, packages and deploys the root stack |
| `docs/` | Discovery report and architecture diagram |

## How the infrastructure is deployed

On every push to `main` that changes `templates/`, GitHub Actions:

1. Gets temporary AWS credentials through OIDC. No keys are stored anywhere.
2. Validates every template.
3. Runs `aws cloudformation package`, which uploads the child templates to the templates bucket and writes their S3 addresses into the root template.
4. Runs `aws cloudformation deploy` on the root stack `onward-dev-ecs`, using the CloudFormation deploy role. CloudFormation builds the children in order: network, service, delivery.

The GitHub Actions role can only upload templates and ask CloudFormation to deploy. It cannot create resources itself.

### Repository variables

| Variable | Value |
| --- | --- |
| `AWS_INFRA_ROLE_ARN` | `InfraActionsRoleArn` output of the bootstrap stack |
| `TEMPLATES_BUCKET` | `TemplatesBucketName` output of the bootstrap stack |
| `CFN_DEPLOY_ROLE_ARN` | The CloudFormation deploy role |
| `CONNECTION_ARN` | The GitHub connection used by CodePipeline |

These are variables, not secrets: none of them grants access on its own. There are no secrets in this repository.

## How the stacks connect

- The **root** reads each child's outputs and passes them to the next child as parameters: network outputs go to service, service outputs go to delivery.
- Values from the **bootstrap** stack (the ECR repository) come from its exports, because bootstrap is a separate stack.

## How a release works

1. An engineer merges a change to `main` in `onward-orders-app`. This is the only manual step.
2. GitHub Actions builds the image, gets temporary AWS credentials through OIDC, and pushes it to ECR as `:<commit>` and `:latest`.
3. EventBridge sees the new `:latest` image and starts CodePipeline.
4. CodePipeline reads `deploy/taskdef.json` and `deploy/appspec.yaml` from the app repo, fills in the exact image, and hands it to CodeDeploy.
5. CodeDeploy starts green tasks, sends 10 percent of traffic to them for 5 minutes, then 100 percent. Blue tasks are kept for 10 minutes so the release can be reversed instantly.

## How to roll back

- **During a release, or within 10 minutes after:** CodeDeploy console, open the deployment, choose **Stop deployment** and **Stop and roll back**. Traffic returns to blue.
- **Later:** revert the commit in `onward-orders-app` and merge. The previous version is released through the same pipeline.

## Capacity

The dev environment runs **1 task** (`MinTasks=1`) so recovery after a failure is quick to demonstrate. It scales up to 6 tasks on average CPU above 50 percent. For production, set `MinTasks=2` so one task runs in each Availability Zone and a failure causes no outage.

## Security

- No long lived credentials. GitHub uses OIDC. AWS services use IAM roles.
- Only the load balancer is public. Tasks run in private subnets with no route to the internet and reach AWS only through VPC endpoints.
- Tasks accept traffic only from the load balancer's security group. No admin ports.
- No inline IAM policies. Roles use AWS managed or customer managed policies.

## Rebuild from scratch

1. Deploy `onward-orders-bootstrap` with Git sync.
2. Push to `main` in `onward-orders-app`, so ECR has a first image.
3. Set the four repository variables in this repo.
4. Push to `main` here, or run **Deploy infrastructure** from the Actions tab.

## Delete everything

1. Empty the artifact bucket `onward-dev-orders-artifacts-<account>`.
2. Delete the root stack `onward-dev-ecs`. It deletes its child stacks in the right order.
3. If removing the bootstrap too: empty the templates bucket (including old versions), then delete `onward-dev-orders-bootstrap`.

## Prerequisites created once (bootstrap exceptions)

- GitHub connection used by CodePipeline and by Git sync for the bootstrap stack
- CloudFormation deploy role `onward-dev-cloudformation-deploy-role`
- Git sync role for the bootstrap stack

## Standards

Names follow `onward-<env>-<purpose>`. Every resource is tagged `Project=onward`, `Owner=Effie Njoki`, `Environment=dev`. Region: `eu-west-1`.