---
layout: instance/blog-post
title: "Terraform state drift detection in GitHub Actions"
tagline: "know your numbers"
featured: false
image: /assets/images/blog/blog-blood-pressure.jpg
---

One of the annoying things to deal with when it comes to Terraform infrastructure is *Terraform state drift*, which describes the **mismatch between the configured Terraform resources and the infrastructure reality**.

<!--more-->

In your code base, you describe your infrastructure using Terraform resources. When ready you have to apply those changes to your infrastructure. Terraform stores applied changes in a Terraform state locally, in Terraform cloud, or in an S3 bucket. But if you do not apply changes or if you manually change your infrastructure outside of Terraform then the Terraform state drifts from your infrastructure code and/or your running infrastructure.

Another cause for Terraform state drift are *ACME certificates*. They expire every three months and need to be renewed. This is reflected in your Terraform state.

## The idea

To detect Terraform state drift early, we are going to check for state drift upon every push to our GitHub repository. To detect manual changes to our infrastructure, we’ll run the state drift check once a day.

![Untitled](/assets/images/blog/terraform-state-drift.png)

For the easier resolution of the state drift, we want the Terraform plan in the GitHub Action run summary (see screenshot). If we have state drift detected, we want to be notified via Slack and the GitHub Action run to fail.

## GitHub Action

The GitHub Action definition is split into four parts:

1. **Set up the Terraform workspace**: After checking out the repository, we can use the `hashicorp/setup-terraform` action to set up a local Terraform installation. As our Terraform state is held in Terraform Cloud, we supply the Terraform Cloud API token as a secret.
2. **Initialize and validate the Terraform configuration**: this will perform the Terraform `init` and `validate`
3. **Execute the Terraform plan** and add the plan output the job summary. The `-detailed-exitcode` lets the process exit with a return code ≠ 0 upon any Terraform changes or problems.
4. **Notify via Slack** if we have detected a Terraform state drift. Also let the GitHub Actions run fail manually to have the failed run recorded in our GitHub Actions dashboard.

{% raw %}
```yaml
name: terraform state drift detection

# Execute this action on push to main and everyday at 7am
on:
  push:
    branches:
      - main
  workflow_dispatch:
  schedule:
    - cron:  '0 7 * * *'

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    # (1) set up workspace, Terraform and
		#     supply credentials needed for the
		#     Terraform plan (DigitalOcean token)
    - uses: actions/checkout@v3
    - uses: hashicorp/setup-terraform@v2
      with:
        cli_config_credentials_token: ${{ secrets.TERRAFORM_CLOUD_API_TOKEN }}
    - name: prepare-credentials
      run: |
        cat << EOF > secrets.auto.tfvars
        do_token = "${{ secrets.DO_TOKEN_RO }}"
        EOF

		# (2) Terraform init and validate
    - id: init
      run: terraform init -no-color -input=false -lock=false
    - id: validate
      run: terraform validate -no-color

		# (3) Execute Terraform plan and add plan to the build summary
    - id: plan
      run: terraform plan -no-color -lock=false -detailed-exitcode -compact-warnings
      continue-on-error: true
    - run: |
        cat << 'EOF' >> $GITHUB_STEP_SUMMARY
        ### 🤖 Terraform plan

        ```terraform
        ${{ steps.plan.outputs.stdout }}
        ```
        EOF

		# (4) Upon Terraform state drift, notify slack and fail build
    - name: Slack Notification
      uses: rtCamp/action-slack-notify@v2
      if: ${{ steps.plan.outputs.exitcode > 0 }}
      env:
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK_URL }}
        SLACK_COLOR: failure
        SLACK_TITLE: Terraform state drift detected
        SLACK_MESSAGE: ":robot: Please check plan for workspace `locations`"
        SLACK_FOOTER: "${{ github.repository }}"
    - name: Fail job on plan changes
      if: ${{ steps.plan.outputs.exitcode > 0 }}
      uses: actions/github-script@v6
      with:
        script: |
            core.setFailed('Terraform state drift detected')
```
{% endraw %}

## Alternatives

There are two alternatives to the workflow above available:

- **Always apply on push**: The straightforward version would be to apply Terraform infrastructure changes straight out of GitHub Actions. This would be a true CI/CD approach for infrastructure.
- **Use [RunAtlantis](https://www.runatlantis.io/) to automate your Pull Requests:** This is a nifty tool to automate infrastructure changes using Pull Requests on either GitHub or Gitlab. Terraform infrastructure changes are published as Pull Requests. The changes and can be applied via Pull Request comment. This gives you a great visual and searchable history of infrastructure changes in your VCS of choice.

But: if someone changes your infrastructure manually outside of your Terraform resources, you’d still have to run a Terraform state drift detection job 🤷‍♂️. Adding the job above to our infrastructure repos at [ping7.io](http://ping7.io) reduced the reaction time for Terraform state drift massively.
