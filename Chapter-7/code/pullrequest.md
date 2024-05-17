# Code example for pull request IaC control

```yaml
name: Pull Requests quality gate

on:
  pull_request:
    branches:
      - main
    paths-ignore:
      - "README.md"

jobs:
  checkov-check:
    runs-on: ubuntu-latest
    name: Check Terraform with Checkov
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Set up Python 3.8
        uses: actions/setup-python@v1
        with:
          python-version: 3.8
      - name: Prepare reports directory
        run: mkdir test-results
      - name: Run Checkov
        id: checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          soft_fail: false
          output_format: junitxml
          output_file_path: test-results
      - name: Publish test artifact
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: Checkov-report
          path: test-results/results_junitxml.xml
      - name: Publish annotations
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          junit_files: "test-results/**/*.xml" 
  plan:
    runs-on: ubuntu-latest
    name: Generate terraform plan
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Login to AWS
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.AWS_CA_ASSUME_ROLE }}
          role-session-name: GitHubActionsSession
          aws-region: eu-west-1
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.3.9
      - name: Prepare environment
        id: tf-init
        run: |
          terraform init -backend-config=backend.hcl
          terraform workspace select poctfmc
      - name: Terraform fmt
        id: tf-fmt
        run: terraform fmt -check
      - name: Terraform validate
        id: tf-validate
        run: terraform validate -no-color
      - name: Create Terraform plan
        id: tf-plan
        run: terraform plan -no-color
      - name: Comment on Pull Request
        uses: actions/github-script@v6
        if: always()
        env:
          PLAN: "terraform\n${{ steps.tf-plan.outputs.stdout }}\nErrors:\n${{ steps.tf-plan.outputs.stderr }}"
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            // 1. Retrieve existing bot comments for the PR
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            })
            const botComment = comments.find(comment => {
              return comment.user.type === 'Bot' && comment.body.includes('Terraform Format and Style')
            })

            // 2. Prepare format of the comment
            const output = `#### Terraform Format and Style 🖌\`${{ steps.tf-fmt.outcome }}\`
            #### Terraform Initialization ⚙️\`${{ steps.tf-init.outcome }}\`
            #### Terraform Validation 🤖\`${{ steps.tf-validate.outcome }}\`
            <details><summary>Validation Output</summary>

            \`\`\`\n
            Output:\n
            ${{ steps.tf-validate.outputs.stdout }}\n
            Errors:\n
            ${{ steps.tf-validate.outputs.stderr }}
            \`\`\`
            

            </details>

            #### Terraform Plan 📖\`${{ steps.tf-plan.outcome }}\`

            <details><summary>Show Plan</summary>

            \`\`\`\n
            ${process.env.PLAN}
            \`\`\`

            </details>

            *Pusher: @${{ github.actor }}, Action: \`${{ github.event_name }}\`, Workflow: \`${{ github.workflow }}\`*`;

            // 3. If we have a comment, update it, otherwise create a new one
            if (botComment) {
              github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: botComment.id,
                body: output
              })
            } else {
              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: output
              })
            }
```
