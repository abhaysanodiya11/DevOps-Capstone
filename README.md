# DevOps Accelerator

A serverless DevOps capstone that accepts payment-confirmation uploads through a static web application and processes them asynchronously on AWS. It demonstrates infrastructure as code, event-driven architecture, and component-specific GitHub Actions deployments.

## Architecture

```text
Browser
  -> S3-hosted frontend delivered through CloudFront
  -> API Gateway HTTP API: POST /generate-presigned-url
  -> Presign Lambda
  -> time-limited S3 upload URL
  -> upload S3 bucket
  -> S3 ObjectCreated event
  -> processing Lambda
  -> SNS topic -> email subscription
```

### Upload flow

1. The frontend requests a presigned upload URL from API Gateway.
2. API Gateway invokes the presign Lambda, which returns a URL for the upload bucket.
3. The browser uploads the selected JPG, PNG, or PDF directly to S3 with that URL.
4. An `s3:ObjectCreated:*` event invokes the processing Lambda.
5. The processing Lambda validates the file extension, writes execution details to CloudWatch Logs, and publishes an SNS notification email.

## AWS services

| Area | Implementation |
| --- | --- |
| Frontend | Static HTML hosted in Amazon S3 and delivered through CloudFront |
| API | API Gateway HTTP API with `POST /generate-presigned-url` |
| Compute | Python Lambda functions for presigning and upload processing |
| Storage | Separate S3 buckets for frontend hosting, file uploads, and Terraform state |
| Notifications | SNS topic with an email subscription |
| Infrastructure | Terraform with an encrypted S3 remote state backend and DynamoDB state locking |
| Observability | Lambda and API Gateway CloudWatch Logs; AWS-managed CloudWatch metrics |

Custom CloudWatch dashboards and CloudWatch alarms are not currently configured.

## CI/CD

GitHub Actions uses AWS credentials stored as repository secrets.

| Workflow | Trigger | Current behavior |
| --- | --- | --- |
| [`backend.yml`](.github/workflows/backend.yml) | Pushes to `main` that change `backend/process-uploaded-file/**`, or manual dispatch | Packages and deploys the processing Lambda |
| [`frontend.yml`](.github/workflows/frontend.yml) | Pushes to `main` that change `frontend/**` | Syncs the frontend to S3 and invalidates CloudFront |
| [`terraform.yml`](.github/workflows/terraform.yml) | Every push to `main` | Runs `terraform init -reconfigure`, `terraform plan`, and `terraform apply -auto-approve` |

Because the Terraform workflow is triggered by every push to `main`, even a frontend or backend-only change currently runs Terraform plan and apply.

## Repository layout

```text
.github/workflows/                 GitHub Actions workflows
backend/
  generate-presigned-url/          Presign Lambda source
  process-uploaded-file/           S3 event-processing Lambda source
frontend/                          Static web application
infra/terraform/                   Terraform configuration, variables, and outputs
```

## Deployment notes

Terraform defines the AWS resources and uses a pre-existing remote backend in S3 with DynamoDB locking. The deployed project has been applied locally against that remote backend as well as through its GitHub Actions workflow; local Terraform apply is therefore not prohibited by the project itself.

Before provisioning a separate deployment, configure AWS credentials, choose globally unique S3 bucket names, create or point Terraform at a suitable state backend, and provide values for the Terraform variables. The workflow also expects these repository secrets:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `FRONTEND_BUCKET_NAME`
- `CLOUDFRONT_DIST_ID`
- `LAMBDA_FUNCTION_NAME`
- `UPLOAD_BUCKET_NAME`

Use Terraform output after an apply to retrieve the current CloudFront domain, frontend bucket name, upload bucket name, processing Lambda name, and API endpoint. The README intentionally does not embed environment-specific URLs or sample output values.
