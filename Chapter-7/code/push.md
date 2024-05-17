# Pipeline example for push

```yaml
name: Example pipeline. GitHub Actions to AWS

on:
  push:
    branches:
      - main
    paths-ignore:
      - "README.md"
env:
  preprod_artifact_name: preprod_tfplan-${{ github.ref_name }}-${{ github.run_id }}-${{ github.run_attempt }}
  prod_artifact_name: prod_tfplan-${{ github.ref_name }}-${{ github.run_id }}-${{ github.run_attempt }}

jobs:
  Validation:
    name: Validate the template
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Login to AWS
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.AWS_CA_ASSUME_ROLE }}
          role-session-name: GitHubActionsSession
          aws-region: eu-central-1
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.4
      - name: Prepare environment
        run: |
          terraform init -backend-config="environments/preprod_backend.hcl"
      - name: Terraform fmt
        run: terraform fmt -check
      - name: Terraform validate
        run: terraform validate -no-color

  Test:
    name: Test the template
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
  
  BuildPreprod:
    name: Build the plan file
    needs: [Validation, Test]
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Login to AWS
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.AWS_CA_ASSUME_ROLE }}
          role-session-name: GitHubActionsSession
          aws-region: eu-central-1
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.4
      - name: Prepare environment
        run: |
          terraform init -backend-config="environments/preprod_backend.hcl"
      - name: Create Terraform plan for Preprod
        run: terraform plan -var-file="environments/preprod.tfvars" -out ${{ env.preprod_artifact_name }}
      - name: Upload artifact for deployment
        uses: actions/upload-artifact@v3
        with:
          name: preprod-execution-plan
          path: ${{ env.preprod_artifact_name }}

  BuildProd:
    name: Build the plan file for Prod
    needs: [Validation, Test]
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Login to AWS
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.AWS_CA_ASSUME_ROLE }}
          role-session-name: GitHubActionsSession
          aws-region: eu-central-1
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.4
      - name: Prepare environment
        run: |
          terraform init -backend-config="environments/prod_backend.hcl"
      - name: Create Terraform plan for Prod
        run: terraform plan -var-file="environments/prod.tfvars" -out ${{ env.prod_artifact_name }}
      - name: Upload artifact for deployment
        uses: actions/upload-artifact@v3
        with:
          name: prod-execution-plan
          path: ${{ env.prod_artifact_name }}

  DeployPreprod:
    name: TF apply on Preprod AWS account
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    needs: [BuildPreprod]
    if: ${{ github.ref }} == "refs/heads/main"
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Login to AWS
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.AWS_CA_ASSUME_ROLE }}
          role-session-name: GitHubActionsSession
          aws-region: eu-central-1
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.4
      - name: Prepare environment
        run: |
          terraform init -backend-config="environments/preprod_backend.hcl"
      - name: Download artifact for deployment
        uses: actions/download-artifact@v3
        with:
          name: preprod-execution-plan
      - name: Execute terraform apply
        run: terraform apply ${{ env.preprod_artifact_name }}

  TestPreprod:
    name: Check Preprod
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    needs: [DeployPreprod]
    if: ${{ github.ref }} == "refs/heads/main"
    steps:
      - name: Login to AWS
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.AWS_CA_ASSUME_ROLE }}
          role-session-name: GitHubActionsSession
          aws-region: eu-central-1
      - name: Run some checks
        run: |
           aws ec2 describe-vpcs --filter Name=tag:Name,Values=preprodvpc

  Approval:
    name: Approval
    runs-on: ubuntu-latest
    permissions:
      issues: write
    needs: [BuildProd, TestPreprod]
    steps:
      - name: Manual approval
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ github.TOKEN }}
          approvers: pawelpiwosz
          minimum-approvals: 1

  DeployProd:
    name: TF apply on Prod AWS account
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    needs: [Approval]
    if: ${{ github.ref }} == "refs/heads/main"
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Login to AWS
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.AWS_CA_ASSUME_ROLE }}
          role-session-name: GitHubActionsSession
          aws-region: eu-central-1
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.4
      - name: Prepare environment
        run: |
          terraform init -backend-config="environments/prod_backend.hcl"
      - name: Download artifact for deployment
        uses: actions/download-artifact@v3
        with:
          name: prod-execution-plan
      - name: Execute terraform apply
        run: terraform apply ${{ env.prod_artifact_name }}

  TestProd:
    name: Check Prod
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    needs: [DeployProd]
    if: ${{ github.ref }} == "refs/heads/main"
    steps:
      - name: Login to AWS
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.AWS_CA_ASSUME_ROLE }}
          role-session-name: GitHubActionsSession
          aws-region: eu-central-1
      - name: Run some checks
        run: |
           aws ec2 describe-vpcs --filter Name=tag:Name,Values=prodvpc

  DestroyPreprod:
    name: TF destroy on Preprod AWS account
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    needs: [Testprod, TestProd]
    if: ${{ github.ref }} == "refs/heads/main"
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Login to AWS
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.AWS_CA_ASSUME_ROLE }}
          role-session-name: GitHubActionsSession
          aws-region: eu-central-1
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.4
      - name: Prepare environment
        run: |
          terraform init -backend-config="environments/preprod_backend.hcl"
      - name: Execute terraform destroy
        run: terraform destroy -auto-approve -var-file="environments/preprod.tfvars"

  DestroyProd:
    name: TF destroy on Prod AWS account
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    needs: [Testprod, TestProd]
    if: ${{ github.ref }} == "refs/heads/main"
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Login to AWS
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.AWS_CA_ASSUME_ROLE }}
          role-session-name: GitHubActionsSession
          aws-region: eu-central-1
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.4
      - name: Prepare environment
        run: |
          terraform init -backend-config="environments/prod_backend.hcl"
      - name: Execute terraform destroy
        run: terraform destroy -auto-approve -var-file="environments/prod.tfvars"
```
