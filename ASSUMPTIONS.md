# Assumptions & Trade-offs

## IaC: Terraform over AWS CDK

The challenge specifies AWS CDK, but Terraform was chosen instead. Terraform is cloud-agnostic, uses a declarative HCL syntax that is easier to review and diff, and has a mature ecosystem of providers. It also enables a clear separation between infrastructure definition and application code.

## Two-module Terraform split

The infrastructure is split into `bootstrap/` and `main/` to solve a real dependency chain problem:

- The ACM certificate requires DNS validation, which requires the Route53 zone to exist and NS records to be delegated from the upstream DNS provider (Cloudflare)
- The ECR repository must exist (and have an image pushed) before ECS can start tasks
- The HTTPS listener depends on a validated ACM certificate

A single `terraform apply` would race between these resources. Splitting into two modules with SSM Parameter Store as the bridge makes the dependency explicit and the deploy process reliable.

## Application

- **Python + FastAPI**: Chosen for simplicity and fast development. The challenge focuses on infrastructure, not application complexity.
- **Multi-stage Docker build**: Keeps the final image small by separating dependency installation from the runtime image.
- **Non-root user**: The container runs as a dedicated `app` user, not root, reducing the blast radius of a container escape.
- **Platform pinned to `linux/amd64`**: ECS Fargate runs on amd64. Without explicit pinning, building on Apple Silicon (arm64) produces an incompatible image.

## Networking

- **Fargate over EC2**: The challenge explicitly requires an ECS service. Within ECS, Fargate was chosen over EC2 launch type to eliminate instance management overhead (no patching, no AMIs, no capacity planning). Trade-off: slightly higher cost per vCPU-hour, but justified for a minimal service with no special OS requirements.
- **2 AZs**: Provides high availability while keeping costs manageable.
- **NAT Gateway per AZ**: Each private subnet has its own NAT Gateway for AZ-independent fault tolerance. A single shared NAT Gateway would save ~$32/month but creates a single point of failure.
- **No public IPs on ECS tasks**: Tasks run in private subnets and are only reachable via the ALB. Outbound traffic goes through NAT Gateways.

## DNS & TLS

- **HTTPS enabled**: TLS termination at the ALB using an ACM certificate validated via DNS. HTTP traffic is redirected to HTTPS with a 301.
- **TLS 1.3 policy**: Uses `ELBSecurityPolicy-TLS13-1-2-2021-06` which supports TLS 1.2 and 1.3 only, dropping older insecure protocols.
- **Domain delegated from Cloudflare**: The Route53 hosted zone is authoritative for `streaver.tinouy.com`, with NS records configured in Cloudflare. This is a pragmatic approach that avoids transferring the entire domain.

## Scaling

- **CPU target 70%**: Leaves headroom before tasks become unresponsive. Standard starting point for compute-bound workloads.
- **Memory target 80%**: Python can spike memory during request processing; 80% avoids premature OOM kills.
- **Request count target 1000/target**: Arbitrary starting value. Should be tuned with load testing in production.
- **Min 2 tasks**: Ensures availability across both AZs at all times. Never scales to 1.
- **Max 6 tasks**: Conservative upper bound to avoid runaway costs. Adjust based on actual traffic patterns.
- **Scale-out cooldown 60s, scale-in cooldown 300s**: Aggressive scale-out to respond to spikes, slow scale-in to avoid flapping.

## Resilience

- **Deployment circuit breaker**: ECS rolls back automatically if new tasks fail to stabilize, preventing bad deployments from taking down the service.
- **Rolling deployment (100% min healthy, 200% max)**: New tasks are started before old ones are drained, ensuring zero-downtime deployments.
- **Health check grace period (30s)**: Gives newly launched tasks time to start before the ALB marks them unhealthy.
- **Task definition ignored in lifecycle**: The ECS service ignores changes to `task_definition` in Terraform, allowing CI/CD to update the image tag without causing Terraform drift.

## Security

- **Least-privilege IAM**: The execution role can only pull from its specific ECR repo and write to its specific log group. The task role has no permissions (the app makes no AWS API calls).
- **Security groups**: ALB accepts port 80 and 443 from the internet; ECS tasks accept traffic only from the ALB security group on the container port.
- **VPC Flow Logs**: Captures rejected traffic for security auditing. Only `REJECT` traffic is logged to minimize costs.
- **ECR scan on push**: Automated vulnerability scanning of container images on every push.
- **ECR image tag mutability**: Tags are mutable to allow `latest` tag updates. In a stricter environment, immutable tags would be preferred with a proper tagging strategy.
- **S3 state encryption**: Terraform state is stored encrypted in S3 with DynamoDB locking to prevent concurrent modifications.
- **IAM access keys for CI**: GitHub Actions uses long-lived IAM access keys stored as repository secrets. This is simpler to set up but less secure than OIDC federation, which would be the production recommendation.

## Observability

- **14-day log retention**: Balances cost with debugging needs. Production systems may want 30-90 days.
- **Container Insights enabled**: Provides CPU, memory, network, and storage metrics at the task level.
- **Five CloudWatch alarms**: High CPU (>85%), high memory (>90%), ALB 5XX errors (>10/min), unhealthy hosts (>0), and low running tasks (<min capacity). Thresholds are conservative defaults that should be tuned after observing baseline behavior.
- **`treat_missing_data = notBreaching`** on 5XX alarm: Avoids false alarms during periods with no traffic.
- **SNS email alerts**: Optional email subscription for alarm notifications. Simple and effective for a small team.

## CI/CD

- **Separate repos for app and infrastructure**: Allows independent lifecycles. App changes (code + Dockerfile) don't require infrastructure changes and vice versa. Each repo has its own CI pipeline.
- **Manual approval for Terraform apply**: The `production` and `destroy` GitHub environments require reviewer approval before applying or destroying infrastructure. This prevents accidental changes from automated pushes.
- **Destroy order**: The destroy workflow destroys main before bootstrap, respecting the dependency chain (main resources reference bootstrap outputs).
- **Image tagged by commit SHA + latest**: The SHA tag enables traceability from deployed container back to source code. The `latest` tag is a convenience for the initial ECS task definition.

## State Management

- **Separate S3 buckets per module**: Each Terraform module has its own state bucket (`streaver-challenge-terraform-bootstrap` and `streaver-challenge-terraform-main`). This provides isolation between bootstrap and main state, reducing the blast radius of state corruption.
- **SSM Parameter Store for cross-module communication**: Bootstrap writes its outputs to SSM, and main reads them as data sources. This avoids the need to pass variables manually between modules and makes the deploy workflow simpler (`terraform apply` with no extra `-var` flags).
- **DynamoDB locking**: Prevents concurrent Terraform runs from corrupting state, especially important with CI/CD pipelines.

