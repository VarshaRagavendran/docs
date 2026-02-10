---
title: Deploying to AWS Elastic Beanstalk
shortTitle: AWS Elastic Beanstalk
intro: Learn how to deploy an application to AWS Elastic Beanstalk using the official GitHub Action as part of a continuous deployment (CD) workflow.
versions:
  fpt: '*'
  ghes: '*'
  ghec: '*'
topics:
  - CD
  - AWS
  - Elastic Beanstalk
---

## Prerequisites

Before you create your {% data variables.product.prodname_actions %} workflow to deploy to AWS Elastic Beanstalk, complete the following setup steps:

1. **Create or identify an Elastic Beanstalk application and environment.**

   You can either:

   - Use an existing Elastic Beanstalk application and environment, or
   - Create a new application and environment from the AWS Management Console or CLI.

   If you are creating the environment for the first time with this action, you must:

   - Decide on an **application name** and **environment name**.
   - Choose a supported **platform** (solution stack name or platform ARN). For example:
     - Solution stack name: `64bit Amazon Linux 2023 v4.3.0 running Python 3.11`
     - Platform ARN: `arn:aws:elasticbeanstalk:us-west-2::platform/Node.js 22 running on 64bit Amazon Linux 2023/6.1.0`
   - Create the required **Elastic Beanstalk IAM roles**:
     - Instance profile (for example, `aws-elasticbeanstalk-ec2-role`)
     - Service role (for example, `aws-elasticbeanstalk-service-role`)

   For more information, see [Managing Elastic Beanstalk instance profiles](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/iam-instanceprofile.html) and [Managing Elastic Beanstalk service roles](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/iam-servicerole.html).

1. **Configure AWS authentication for GitHub Actions.**

   You can authenticate from GitHub to AWS in one of two ways:

   - **OpenID Connect (OIDC) with an IAM role** (recommended), or
   - **Long‑lived access keys** for an IAM user.

   {% data reusables.actions.about-oidc-short %}

   For OIDC, create:

   - An IAM identity provider for `https://token.actions.githubusercontent.com`.
   - An IAM role that trusts your {% data variables.product.prodname_dotcom %} repository, and attach the required Elastic Beanstalk and S3 permissions.

   For more information, see the official [Configure AWS Credentials](https://github.com/aws-actions/configure-aws-credentials) action documentation.

1. **Attach required IAM permissions.**

   The role that GitHub Actions assumes must be able to:

   - Call Elastic Beanstalk APIs to create and manage application versions and environments.
   - Upload deployment bundles to Amazon S3 for use as Elastic Beanstalk application versions.

   The simplest option is to attach the AWS managed policy **`AdministratorAccess-AWSElasticBeanstalk`** to the role. For production environments, you can alternatively use a scoped custom policy that grants:

   - Elastic Beanstalk operations (`elasticbeanstalk:*`), and
   - S3 access to the deployment bucket used for application versions.

   The deployment bucket defaults to `elasticbeanstalk-{region}-{accountId}` (for example, `elasticbeanstalk-us-west-2-123456789012`), unless you set a custom bucket using the `s3-bucket-name` input in the workflow.

   At a minimum, the caller role must be able to perform the following S3 actions on the deployment bucket:

   - `s3:GetObject`
   - `s3:GetObjectVersion`
   - `s3:CreateBucket`
   - `s3:ListBucket`
   - `s3:GetBucketLocation`
   - `s3:GetBucketAcl`
   - `s3:PutObject`

1. **Create GitHub secrets.**

   Create the following secrets in your repository (Settings → Secrets and variables → Actions):

   - If using OIDC:
     - `AWS_ROLE_TO_ASSUME`: The ARN of the IAM role to assume from GitHub Actions.
   - If using static credentials:
     - `AWS_ACCESS_KEY_ID`: The access key ID for your IAM user.
     - `AWS_SECRET_ACCESS_KEY`: The secret access key for your IAM user.

1. **Optionally, configure a deployment environment.** {% data reusables.actions.about-environments %}

## Creating the workflow

Once you have completed the prerequisites, you can create a workflow to build and deploy your application to Elastic Beanstalk.

The following example shows how to:

- Configure AWS credentials.
- Build (or use) a deployment package.
- Create or reuse an Elastic Beanstalk application version.
- Create or update an Elastic Beanstalk environment.

Create a workflow file in your repository at `.github/workflows/deploy-to-elastic-beanstalk.yml` (or any name you prefer under `.github/workflows/`), then use the following as a starting point.

{% data reusables.actions.delete-env-key %}

```yaml copy
{% data reusables.actions.actions-not-certified-by-github-comment %}

{% data reusables.actions.actions-use-sha-pinning-comment %}

name: Deploy to AWS Elastic Beanstalk

on:
  push:
    branches:
      - main

env:
  AWS_REGION: us-west-2                         # Set this to your preferred AWS region
  APPLICATION_NAME: my-app                      # Elastic Beanstalk application name
  ENVIRONMENT_NAME: my-app-prod                 # Elastic Beanstalk environment name

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    environment: production
    permissions:
      id-token: write
      contents: read

    steps:
      - name: Checkout repository
        uses: {% data reusables.actions.action-checkout %}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@0e613a0980cbf65ed5b322eb7a1e075d28913a83
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Deploy to Elastic Beanstalk
        uses: aws-actions/aws-elasticbeanstalk-deploy@<sha>
        with:
          aws-region: ${{ env.AWS_REGION }}
          application-name: ${{ env.APPLICATION_NAME }}
          environment-name: ${{ env.ENVIRONMENT_NAME }}

          # Platform configuration (required when creating a new environment)
          # Provide either solution-stack-name OR platform-arn:
          # solution-stack-name: "64bit Amazon Linux 2023 v4.3.0 running Python 3.11"
          platform-arn: "arn:aws:elasticbeanstalk:${{ env.AWS_REGION }}::platform/Node.js 22 running on 64bit Amazon Linux 2023/6.1.0"

          # Ensure the application and environment are created if they do not exist
          create-application-if-not-exists: true
          create-environment-if-not-exists: true

          # Option settings must include the Elastic Beanstalk IAM instance profile and service role
          option-settings: |
            [
              {
                "Namespace": "aws:autoscaling:launchconfiguration",
                "OptionName": "IamInstanceProfile",
                "Value": "aws-elasticbeanstalk-ec2-role"
              },
              {
                "Namespace": "aws:elasticbeanstalk:environment",
                "OptionName": "ServiceRole",
                "Value": "aws-elasticbeanstalk-service-role"
              }
            ]
```

### Reusing existing application versions

Elastic Beanstalk uses **application versions** to identify a specific deployment bundle stored in Amazon S3. The `aws-elasticbeanstalk-deploy` action can either **reuse** an existing version or **create** a new one, depending on the `use-existing-application-version-if-available` and `version-label` inputs.

- When `use-existing-application-version-if-available` is `true` (the default):
  - If an application version with the specified `version-label` already exists, the action **reuses** that version and does not re-upload or recreate it.
  - If it does not exist, the action creates a **new** application version using the deployment package.

- When `use-existing-application-version-if-available` is `false`:
  - The action always attempts to **create a new** Elastic Beanstalk application version for the specified `version-label`.
  - If a version with that label already exists, the deployment fails fast with an error such as `Application Version <label> already exists.`
  - In this mode, you must ensure that `version-label` is **unique per deployment**.

For example, you can use the commit SHA as a version label:

```yaml copy
with:
  version-label: ${{ github.sha }}
  use-existing-application-version-if-available: true
```

This pattern works well when deploying the **same application version** to multiple Elastic Beanstalk environments (for example, staging and production) from the same build.

## Further reading

For more information on the services and actions used in this example, see:

* [AWS Elastic Beanstalk Developer Guide](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/)
* [Supported Elastic Beanstalk platforms](https://docs.aws.amazon.com/elasticbeanstalk/latest/platforms/platforms-supported.html)
* [Managed platform updates](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/environment-platform-update-managed.html)
* Official AWS [Configure AWS Credentials](https://github.com/aws-actions/configure-aws-credentials) action.
* Official AWS [Elastic Beanstalk deploy](https://github.com/aws-actions/aws-elasticbeanstalk-deploy) action.

