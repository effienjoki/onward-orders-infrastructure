# Discovery Report: Delivering Order System Updates Safely for Onward Logistics

| **Lab** | Delivering Order System Updates Safely for Onward Logistics |
| --- | --- |
| **Name** | Effie Njoki |
| **Date** | 5 November 2026 |
| **Client** | Onward Logistics (see the Client Profile) |

## 1. What I found

### The client's need

Onward needs every approved change to its order system to reach customers automatically and without downtime, with a quick way back if a release goes wrong, while keeping the application off the internet and keeping cloud credentials out of its code and pipelines.

### Solutions that exist

| **Solution** | **How it works** | **Trade off for this client** |
| --- | --- | --- |
| Manual releases | An engineer builds the image on a laptop, uploads it, and updates the running service by hand. | Strength: nothing to build. Weakness: slow, depends on one person getting every step right, needs credentials on laptops, and nobody wants to do it on a peak day. This is what Onward does today. |
| Automated pipeline with rolling updates | A pipeline builds and pushes the image, then the service replaces old containers with new ones a few at a time. | Strength: automatic and simple. Weakness: old and new versions serve customers at the same time during the swap, and going back means running a second release. |
| Automated pipeline with blue/green deployment | A pipeline builds and pushes the image. A new set of containers (green) starts beside the current set (blue), traffic moves to green only once it is healthy, and blue is kept for a while. | Strength: no downtime, and going back is a traffic switch. Weakness: more moving parts, and two sets of containers run briefly during each release. |

### What happens with no solution

Manual releases fail in predictable ways. In 2012, Knight Capital deployed new code by hand to eight servers and a technician missed one. When the market opened, its system sent millions of unintended orders in about 45 minutes, and the firm lost more than 460 million US dollars. The SEC later found it lacked adequate controls and procedures for code deployment and testing. One missed manual step nearly ended the company.

Long lived credentials are the second risk. In 2016, attackers broke into Uber engineers' GitHub accounts, found an AWS access key stored in plain text in a private repository, and used it to download files containing personal data. The FTC cited Uber's handling of these keys in an expanded settlement.

For Onward, a broken release on a sales day means failed orders and lost merchants, and a leaked key could expose customer and merchant data, which is the company's product.

### Sources

- SEC press release, Knight Capital: https://www.sec.gov/news/press-release/2013-222
- FTC, Uber revised complaint analysis: https://www.ftc.gov/es/system/files/documents/cases/1523054_uber_technologies_revised_analysis.pdf
- GitHub Docs, Configuring OpenID Connect in AWS: https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
- AWS re:Post, VPC endpoints required for Fargate in private subnets: https://www.repost.aws/knowledge-center/ecs-fargate-tasks-pending-state
- AWS, Amazon ECS blue/green deployments with CodeDeploy: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html
- AWS PrivateLink pricing: https://aws.amazon.com/privatelink/pricing/

## 2. The business value of my solution

### My chosen approach

I chose an automated pipeline with blue/green deployment, because it is the only option that gives the CTO both zero downtime releases and a fast way back.

How a change flows: an engineer merges to main, which is the only manual step. GitHub Actions gets temporary AWS credentials through OIDC, builds the image and pushes it to Amazon ECR. Amazon EventBridge sees the push and starts AWS CodePipeline. CodePipeline hands the new image to AWS CodeDeploy, which starts green containers on ECS Fargate, shifts 10 percent of traffic to them first, then the rest, and keeps blue running for a short window so a bad release can be reversed by switching traffic back.

Key design choices:

- **Three repositories.** Bootstrap (ECR repository, OIDC provider, GitHub role), infrastructure (network, ECS service, delivery pipeline) and application (code, Dockerfile, workflow). Each has one job, the application pipeline can only push images, and a new engineer can see where everything lives.
- **Smaller templates.** The infrastructure is split into network, service and delivery stacks, connected by exports and imports. Each is short enough to read. I did not use nested stacks because their child templates must be uploaded to S3, and Git sync deploys straight from the repository.
- **No internet path.** Tasks run in private subnets with no NAT gateway and no default route. They reach AWS through VPC endpoints: interface endpoints for ECR API, ECR Docker and CloudWatch Logs, and a free S3 gateway endpoint for image layers.
- **No long lived credentials.** GitHub uses OIDC and a role limited to one repository and branch. AWS services use IAM roles.
- **Capacity.** Minimum 2 tasks, one per Availability Zone, maximum 6, scaling on average CPU near 50 percent. Each task is small (0.25 vCPU, 512 MB).

**Private connectivity versus a NAT gateway.** Interface endpoints are charged per endpoint, per Availability Zone, per hour, plus data processed. Three endpoints in two zones means six endpoint hours every hour. The S3 gateway endpoint is free. A NAT gateway was rejected because the brief forbids it and it would give the application an internet path. My cost estimate for normal and busy days is here: [AWS Pricing Calculator estimate link]

**Idle cost.** The service can scale to zero tasks, but endpoints and the load balancer still charge by the hour. When an environment is idle, deleting its stacks stops all charges, and Git sync rebuilds them from the repositories.

### Services I will use

| **Service** | **The job it does** | **Why this service, used this way** |
| --- | --- | --- |
| Amazon VPC | Public subnets for the load balancer, private subnets for tasks, in two Availability Zones. | Two zones for resilience. Private subnets have no internet route. |
| VPC endpoints | Private connections to ECR, CloudWatch Logs and S3. | Keeps traffic to AWS on AWS's network, as the Head of Security requires. |
| Application Load Balancer | The one public entry point, with two target groups for blue and green. | Lets CodeDeploy shift traffic between versions without downtime. |
| Amazon ECS on Fargate | Runs the order containers. | No servers to patch or size, which suits a small platform team. |
| Application Auto Scaling | Adds and removes tasks based on CPU. | Grows on busy days and shrinks afterwards without anyone acting. |
| Amazon ECR | Stores images, with scan on push and a lifecycle rule. | Private, controlled by IAM, scans for known vulnerabilities, deletes old images to control cost. |
| GitHub Actions with OIDC | Builds and pushes the image on every merge. | No stored keys. The role can only push to one repository. |
| Amazon EventBridge | Starts the pipeline when a new image is pushed. | Event driven, so nothing polls and nobody presses a button. |
| AWS CodePipeline | Connects the new image to the deployment. | Gives each release a visible, recorded history. |
| AWS CodeDeploy | Blue/green deployment to ECS, canary 10 percent first, automatic rollback on failure. | Zero downtime and a quick way back, as the CTO asked. |
| Amazon CloudWatch | Container logs and CPU metrics. | Operations can see what happened without logging in anywhere. |
| AWS CloudFormation with Git sync | Builds all infrastructure from the repositories. | Repeatable and rebuildable, matching company standards. |

### Who benefits, and how I will know

| **Stakeholder** | **Benefit** | **How I would know** |
| --- | --- | --- |
| CTO | Merging is the only manual step, releases cause no outage, and bad releases can be reversed. | A merge produces a new version on the page with no other action. During the release, the page keeps loading. A rollback restores the previous version. |
| Head of Security | No long lived credentials, one public entry point, no internet path for the application. | No access keys exist in any repository or pipeline. Task subnets have no route to the internet. |
| Head of Operations | Survives a failed container or a lost zone, and grows on busy days without anyone acting. | A stopped task is replaced automatically. Under load, task count rises above 2, then returns to 2. |
| Finance Lead | Understands the cost of private connectivity, and idle environments can be switched off. | A shared Pricing Calculator estimate, and stacks that can be deleted and rebuilt from the repositories. |
| New engineers | Can understand the setup without a guided tour. | Each repository has a README explaining its job and how it connects to the others. |

### Limits and risks of my design

- **Endpoints cost money even when idle.** Interface endpoints are charged hourly whether or not traffic flows. Idle environments should be deleted, not just scaled down.
- **Infrastructure changes to the running service go through CodeDeploy.** Because CodeDeploy controls deployments, a new task definition is released through the pipeline, not by editing the template.
- **Orders are not stored in a shared database.** Orders are held in memory per container and are lost when a container is replaced. Production would need a database.
- **No HTTPS.** The lab has no domain or certificate. Production would need HTTPS.
- **The first image must exist before the service.** The application pipeline must push an image before the service stack is created.
- **One Region only.** The design survives the loss of a zone, not a whole Region.