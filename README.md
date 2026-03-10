# Cost-Effective GitHub Advanced Security (GHAS) Implementation Guide
## Executive Summary
Rolling out GitHub Advanced Security (GHAS) globally across an enterprise can be a significant upfront investment. However, with strategic planning, you can optimize the consumption of your GHAS licenses.
Because GHAS is billed per **Unique Active Committer** (and not per repository), a developer who commits to 10 GHAS-enabled repositories only consumes **one** license. Furthermore, if a user does not commit to a GHAS-enabled repository for 90 days, their license is freed.
By leveraging **Security Configurations**, **Custom Properties**, and **Dynamic Automation**, we can automate a phased rollout that targets repositories with high developer overlap, effectively scaling your security coverage at $0 additional cost.
## Step 1: Establish Your Baseline Security Configuration
Security configurations simplify the rollout of GHAS at scale by defining collections of settings that can be applied to groups of repositories automatically.
1. Navigate to your Organization page on GitHub.
2. Go to **Settings** > **Code security and analysis** > **Security configurations**.
3. Click **New configuration**.
4. Name it `GHAS Standard Enforcement.`
5. Enable the following features:
   * **Dependabot alerts** & **Dependabot security updates**
   * **Secret scanning** & **Push protection**
   * **Code scanning** (Default setup)
6. **CRITICAL:** Do *not* apply this to "All repositories" or "Newly created repositories." We want granular, automated control based on custom properties. Click **Save configuration**.
7. Look at the URL in your browser and note the configuration ID at the end (e.g., `…/configurations/12345`). You will need this for the automation script.

# Step 2: Establish Phase Control (Properties & Variables)
Instead of hardcoding rules into scripts, we use Organization Variables and Custom Properties to control the rollout dynamically from the GitHub UI.
1. **Create the Property:** In Org **Settings** > **Custom properties**, add a `Single select` property named `ghas_rollout_phase` with the following allowed values: `pilot`, `phase-1-overlap`, `phase-2`, `disabled`.
2. **Create the Control Variable:** Go to Org **Settings** > **Variables** > **Actions**. Create an Organization Variable named `APPROVED_GHAS_PHASES`. Set its value to: `pilot`,`phase-1-overlap`. *(Note: When you are ready to expand to phase 2, you simply update this variable to* *pilot,phase-1-overlap,phase-2* *without needing to modify any automation code).*

# Step 3: Identify & Bulk-Tag Repositories via IssueOps
Before enabling GHAS broadly, run the open-source [ghas-license-utilization](https://github.com/advanced-security/ghas-license-utilization) script to find repositories where 100% of the active committers *already* have a GHAS license.
To easily tag these repositories (or onboard future ones), we will use an **IssueOps** workflow. This creates a self-service UI for managers to submit a list of repositories and select their phase, automatically generating an audit trail.
### 3A: Create the Issue Form Template
In your central management repository, create a file at .github/ISSUE_TEMPLATE/bulk-tag.yml. This creates the form UI.
```
name: Bulk Tag Repositories
description: Apply a GHAS rollout phase to multiple repositories
title: "[Bulk Tag]: "
labels: ["bulk-tag"]
body:
  - type: dropdown
    id: phase
    attributes:
      label: Phase
      description: Select the rollout phase property to apply
      options:
        - pilot
        - phase-1-overlap
        - phase-2
        - disabled
    validations:
      required: true
  - type: textarea
    id: repositories
    attributes:
      label: Repositories
      description: List the exact repository names (one per line)
    validations:
      required: true
```
### 3B: Create the IssueOps Action
Create a file at .github/workflows/issueops-bulk-tag.yml. This Action runs when the form is submitted, parses the markdown, tags the repos, and closes the issue.
name: IssueOps - Bulk Tag Repositories
```
name: IssueOps - Bulk Tag Repositories

on:
  issues:
    types: [opened]

jobs:
  bulk-tag:
    # Only run if the issue has the specific IssueOps label
    if: contains(github.event.issue.labels.*.name, 'bulk-tag')
    runs-on: ubuntu-latest
    steps:
      - name: Generate Token from GitHub App
        id: generate_token
        uses: actions/create-github-app-token@v2.2.1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}

      - name: Parse Issue Body
        id: parse
        run: |
          # Save issue body to a file securely
          cat << 'EOF' > issue_body.txt
          ${{ github.event.issue.body }}
          EOF

          # Extract Phase selection
          PHASE=$(awk '/### Phase/{getline; getline; print}' issue_body.txt | tr -d '\r' | xargs)
          echo "phase=$PHASE" >> $GITHUB_OUTPUT

          # Extract Repository List
          awk '/### Repositories/{flag=1; next} /^###/{flag=0} flag {print}' issue_body.txt | tr -d '\r' > repo_list.txt

      - name: Apply Custom Properties
        env:
          GH_TOKEN: ${{ steps.generate_token.outputs.token }}
          PHASE: ${{ steps.parse.outputs.phase }}
        run: |
          echo "Applying phase: $PHASE"
          
          while IFS= read -r repo; do
            repo=$(echo "$repo" | xargs)
            if [ -n "$repo" ]; then
              echo "Tagging $repo as $PHASE..."
              gh api \
                --method PATCH \
                -H "Accept: application/vnd.github+json" \
                -H "X-GitHub-Api-Version: 2022-11-28" \
                /repos/${{ github.repository_owner }}/$repo/properties/values \
                -f properties[][property_name]="ghas_rollout_phase" \
                -f properties[][value]="$PHASE"
            fi
          done < repo_list.txt

      - name: Close Issue & Comment
        env:
          GH_TOKEN: ${{ steps.generate_token.outputs.token }}
        run: |
          gh issue comment ${{ github.event.issue.number }} --repo ${{ github.repository }} -b "✅ **Automation Complete!** Successfully tagged repositories with the \`${{ steps.parse.outputs.phase }}\` property."
          gh issue close ${{ github.event.issue.number }} --repo ${{ github.repository }}
```
## Step 4: Automate the Rollout (Org-Wide Enablement)
Once a repository is tagged (either manually or via the IssueOps form), we need an Organization Webhook to pass that data to a centralized GitHub Action, which will apply the Security Configuration.
### 4A: Create the Configuration Workflow
Create a file at .github/workflows/ghas-auto-enabler.yml in your management repository.
```
name: Enforce GHAS Security Configuration

on:
  workflow_dispatch:
    inputs:
      target_repository:
        description: 'The name of the repository to evaluate (e.g., my-repo)'
        required: true
        type: string

jobs:
  check-and-enable:
    runs-on: ubuntu-latest
    steps:
      - name: Generate Token from GitHub App
        id: generate_token
        uses: actions/create-github-app-token@v2.2.1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}

      - name: Fetch Repository Properties
        id: get_props
        env:
          APP_TOKEN: ${{ steps.generate_token.outputs.token }} 
          TARGET_REPO: ${{ inputs.target_repository }}
        run: |
          # Fetch using curl to explicitly enforce App Token usage and parse with jq.
          # The .[]? syntax ensures it doesn't crash if the repo has zero properties assigned.
          PHASE=$(curl -s -H "Accept: application/vnd.github+json" \
            -H "Authorization: Bearer $APP_TOKEN" \
            -H "X-GitHub-Api-Version: 2022-11-28" \
            https://api.github.com/repos/${{ github.repository_owner }}/$TARGET_REPO/properties/values \
            | jq -r '.[]? | select(.property_name=="ghas_rollout_phase") | .value')
          
          echo "phase=$PHASE" >> $GITHUB_OUTPUT

      - name: Apply Security Configuration
        # Dynamically checks if the repo's phase is currently in the Approved Phases Organization Variable
        if: contains(vars.APPROVED_GHAS_PHASES, steps.get_props.outputs.phase) && steps.get_props.outputs.phase != ''
        env:
          APP_TOKEN: ${{ steps.generate_token.outputs.token }}
          CONFIG_ID: '238329' # Replace with your Configuration ID from Step 1
          TARGET_REPO: ${{ inputs.target_repository }}
          ORG_NAME: ${{ github.repository_owner }}
        run: |
          echo "Repository is in an approved phase ($TARGET_REPO). Enabling GHAS..."
          
          # The attach API requires a numeric Repository ID, not a string name
          REPO_ID=$(curl -s -H "Accept: application/vnd.github+json" \
            -H "Authorization: Bearer $APP_TOKEN" \
            -H "X-GitHub-Api-Version: 2022-11-28" \
            https://api.github.com/repos/$ORG_NAME/$TARGET_REPO | jq -r '.id')
          
          # Issue a POST request to attach the configuration to the specific ID
          curl -s -X POST \
            -H "Accept: application/vnd.github+json" \
            -H "Authorization: Bearer $APP_TOKEN" \
            -H "X-GitHub-Api-Version: 2022-11-28" \
            https://api.github.com/orgs/$ORG_NAME/code-security/configurations/$CONFIG_ID/attach \
            -d '{"scope":"selected","selected_repository_ids":['"$REPO_ID"']}'
```
### 4B: Set up the Webhook Bridge
Because native GitHub Webhooks cannot inject an Authorization: Bearer header, you must configure a lightweight proxy (e.g., AWS Lambda, Azure Function, or an integration tool like Make/Zapier) to bridge the gap:
1. Create an **Organization Webhook** listening to Repository events (specifically edited / property changes).
2. Point the payload to your Proxy URL.
3. Have your Proxy extract the repository.name from the webhook payload and execute the following API call to trigger your Action:
```
curl -L \
  -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer <YOUR_ORG_ADMIN_PAT>" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  [https://api.github.com/repos/YOUR_ORG/org-management-automation/actions/workflows/ghas-auto-enabler.yml/dispatches](https://api.github.com/repos/YOUR_ORG/org-management-automation/actions/workflows/ghas-auto-enabler.yml/dispatches) \
  -d '{"ref":"main","inputs":{"target_repository":"extracted-repo-name"}}'
```
*(Note for Enterprise Architects: Creating an internal* ***GitHub App*** *subscribed to* *repository* *events is the official enterprise best practice to listen for org-wide events and make API calls without managing a separate proxy server).*
## Step 5: Managing Pilot Repos & Cost Spikes
A GHAS license is consumed when a commit is pushed to a GHAS-enabled repository. It is freed after 90 days of inactivity.
* **Sandbox/POC Repositories:** If a repository was purely used to test GHAS (a sandbox) during the pilot phase, you should detach the Security Configuration and archive the repository when the test is complete. Those licenses will free up automatically after 90 days of no commits.
* **Production Repositories:** If a pilot repository is a real production app and experiences a massive influx of new developers, it *will* consume new licenses. It is an anti-pattern to turn off security on active production code just to save money.
* **Cost Governance:** Control costs by strictly managing *entry* via the **IssueOps** process. Only approve repositories that fit the budget or have high committer overlap. Once enabled, rely on the **Enterprise Billing Dashboard** to monitor Active Committer counts.

# Step 5: Optimizing Code Scanning (Targeting specific branches)
By default, the "Default Setup" applied in Step 1 will scan your main branch **and** all pull requests targeting your protected branches. While excellent for security, running CodeQL on every pull request can consume significant GitHub Actions minutes.
If your goal is to minimize compute costs and **only** scan the main branch, you must use **Advanced Setup** for Code Scanning instead of Default Setup.
To do this:
1. 1	Ensure "Code scanning (Default setup)" is set to **Disabled** in your Organization Security Configuration (Step 1).
2. 2	Push a .github/workflows/codeql.yml file to the repositories you wish to scan.
3. 3	Explicitly define the on: triggers in the YAML file to only target the main branch, completely removing the pull_request trigger:

```
name: "CodeQL Advanced (Main Only)"

on:
  push:
    branches: [ "main" ]
  schedule:
    - cron: '20 4 * * 1' # Runs a weekly scan as a best practice

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Initialize CodeQL
      uses: github/codeql-action/init@v3
      with:
        languages: 'javascript, python' # Update with your repo's languages

    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v3
```
*This approach ensures that developers get immediate feedback in the Security tab after merging to main, without burning GitHub Actions minutes during the iterative Pull Request process.*
