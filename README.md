Terraform AWS CI/CD

A hands-on DevOps project that deploys a containerized Node.js application to AWS EC2 using Terraform, GitHub Actions, Docker, and Amazon ECR.

Built by following and troubleshooting a [CI/CD with Terraform and AWS tutorial](https://www.youtube.com/watch?v=5sZAx2ylsOo), with updates for the AWS account setup and application runtime used in this project.

## Deployment flow

```text
Push to GitHub
    |
    v
GitHub Actions: deploy-infra
    |-- Initialize Terraform with S3 remote state
    |-- Plan and apply infrastructure changes
    |-- Export the EC2 public IP
    |
    v
GitHub Actions: deploy-app
    |-- Build the Docker image
    |-- Push the image to Amazon ECR
    |-- Connect to EC2 over SSH
    |-- Pull and run the new image
    |
    v
HTTP request -> EC2 port 80 -> container port 8080 -> Node.js app
```

The deployed application returns `Service is up and running`.

## Technologies

- **Node.js and Express:** application and HTTP endpoint
- **Docker:** application packaging and container execution
- **Terraform:** infrastructure as code
- **GitHub Actions:** automated deployment workflow
- **Amazon EC2:** Ubuntu application server
- **Amazon ECR:** private Docker image repository
- **Amazon S3:** Terraform remote state
- **AWS IAM:** deployment permissions and the EC2 role
- **SSH:** remote deployment authentication

## Project structure

```text
.github/
  workflows/
    deploy.yml
nodeapp/
  app.js
  Dockerfile
  package.json
  package-lock.json
terraform/
  main.tf
  variables.tf
.gitignore
README.md
```

## Infrastructure

The Terraform configuration manages:

- An Ubuntu EC2 instance using `t3.micro` in `us-west-1`.
- A security group with inbound SSH on port 22 and HTTP on port 80.
- An EC2 key pair created from the supplied public SSH key.
- An instance profile associated with the existing `EC2-ECR-Auth` IAM role.

The S3 state bucket, ECR repository, IAM deployment user, and EC2 IAM role are created separately. They are not all provisioned by this Terraform configuration.

Terraform stores state at:

```text
s3://<your-state-bucket>/aws/ec2-deploy/terraform.tfstate
```

## Setup

### Prerequisites

- An AWS account and access to the AWS Console.
- A GitHub repository with Actions enabled.
- Git and Terraform installed locally for development.
- An SSH key pair for EC2 access.
- An Ubuntu x86_64 AMI available in `us-west-1`.

### AWS resources and permissions

1. Create a private S3 bucket for Terraform state in `us-west-1`. Keep public access blocked, enable versioning, and use server-side encryption.
2. Create a private ECR repository named `example-node-app` in `us-west-1`, or update both repository references in the workflow to match your chosen name.
3. Create the `EC2-ECR-Auth` role with EC2 as its trusted service and permission to pull images from ECR.
4. Configure an IAM deployment user with permissions for the S3 state, required EC2 operations, project instance-profile management, passing the EC2 role, and ECR image uploads.
5. Confirm the target Region has the default VPC/subnet expected by the configuration, or explicitly configure networking in Terraform.

The workflow uses long-lived IAM access keys as part of this learning implementation. A future improvement is GitHub OIDC with an AWS role and temporary credentials.

### GitHub repository secrets

Under **Settings -> Secrets and variables -> Actions -> Repository secrets**, configure:

| Secret                     | Value                                                            |
| -------------------------- | ---------------------------------------------------------------- |
| `AWS_ACCESS_KEY_ID`        | Deployment IAM user's access key ID                              |
| `AWS_SECRET_ACCESS_KEY`    | Matching AWS secret access key                                   |
| `AWS_TF_STATE_BUCKET_NAME` | Exact S3 bucket name, without `s3://` or a path                  |
| `AWS_SSH_KEY_PRIVATE`      | Complete private SSH key, including BEGIN/END lines              |
| `AWS_SSH_KEY_PUBLIC`       | Complete public SSH key line, including `ssh-rsa` when using RSA |

Never commit private keys, AWS credentials, Terraform state, or secret variable files. Keep `.terraform.lock.hcl` and `package-lock.json` in version control.

### Configure and deploy

1. Set the EC2 AMI ID to an appropriate Ubuntu x86_64 image in `us-west-1`.
2. Confirm the workflow's Region, ECR repository name, IAM role name, and Terraform variables match your AWS resources.
3. Use a supported Node.js Docker base image. This project was updated to `node:24-bookworm-slim` to resolve a dependency/runtime incompatibility.
4. Push changes to the workflow's configured branch:

```text
Complete-CI/CD-with-Terraform-AWS
```

5. Open the repository's **Actions** tab and inspect both `deploy-infra` and `deploy-app`.
6. When deployment completes, open:

```text
http://<current-ec2-public-ip>
```

A successful workflow does not by itself prove the application is healthy. Verify the HTTP response and container status.

## Useful checks

Format and validate Terraform from the `terraform` directory after initialization:

```sh
terraform fmt
terraform validate
```

On the EC2 instance:

```sh
sudo docker ps -a
sudo docker logs --tail 50 myappcontainer
curl -v http://localhost:80
```

The workflow maps host port `80` to application port `8080`. The application must listen on port 8080 and accept connections from outside the container's loopback interface.

## Troubleshooting and lessons learned

| Issue encountered                                            | Resolution                                                                                           |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------- |
| Workflow did not run                                         | Pushed to the branch configured in the workflow trigger.                                             |
| Empty S3 bucket configuration                                | Matched the GitHub secret reference to `AWS_TF_STATE_BUCKET_NAME`.                                   |
| S3 or resource creation returned AccessDenied                | Added the required IAM permissions to the deployment user.                                           |
| Invalid HTTP security-group rule                             | Used TCP for port 80; reserved protocol `-1` with ports 0 for all-protocol egress.                   |
| EC2 rejected the instance type under the account's Free plan | Switched from `t2.micro` to `t3.micro`. Eligibility depends on the account and current AWS offering. |
| SSH action reported a missing server host                    | Read the IP from the Terraform directory and passed it between jobs using `$GITHUB_OUTPUT`.          |
| Dockerfile failed to parse                                   | Corrected `EXPORT` to `EXPOSE`.                                                                      |
| Docker push reported a missing repository                    | Created the matching ECR repository in the target Region.                                            |
| Container exited with `Object.hasOwn is not a function`      | Updated the application's Docker Node.js runtime.                                                    |

## Cost management and cleanup

AWS resources can consume credits or incur charges while this project exists. Free Tier eligibility does not mean unlimited free usage.

- For a short break, stop the EC2 instance. Compute usage stops, but attached EBS storage and other stored resources can continue to incur usage.
- Starting the instance again can change its public IPv4 address. The current Docker command does not configure an automatic restart policy, so the application container may need to be started or redeployed after a reboot.
- To remove Terraform-managed infrastructure, initialize Terraform against the **same S3 backend** and supply the same required variables, then review a destroy plan:

```sh
terraform plan -destroy -out=destroy.tfplan
terraform apply destroy.tfplan
```

The second command deletes the resources in that plan. Store plan files securely and do not commit them. Keep the state bucket until destruction has completed successfully.

ECR images/repositories and the separately created S3 bucket are not removed by this Terraform configuration. Review them separately, including old S3 object versions, EBS volumes, snapshots, and any allocated Elastic IP addresses. Check AWS Billing and Cost Management to confirm remaining usage.

## Improvements to explore

- Replace IAM user access keys with GitHub OIDC.
- Use the EC2 instance role for ECR pulls instead of forwarding AWS user credentials over SSH.
- Restrict SSH access and evaluate Systems Manager for remote operations.
- Add application tests and a deployment health check that fails the workflow if the app is unavailable.
- Configure container restart behavior.
- Pin third-party actions to reviewed commit SHAs.
- Add HTTPS and a domain name.
- Provision supporting resources through a separate Terraform bootstrap configuration.
- Add a controlled cleanup workflow and deployment concurrency protection.

## Credits

Original learning resource: [Complete CI/CD with Terraform and AWS tutorial](https://www.youtube.com/watch?v=5sZAx2ylsOo).

This repository documents my hands-on implementation, troubleshooting, and updates while learning infrastructure as code and automated application delivery.
