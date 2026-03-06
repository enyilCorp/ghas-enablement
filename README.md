# Cost-Effective GitHub Advanced Security (GHAS) Implementation Guide
# Executive Summary
Rolling out GitHub Advanced Security (GHAS) globally across an enterprise can be a significant upfront investment. However, with strategic planning, you can optimize the consumption of your GHAS licenses.
Because GHAS is billed per **Unique Active Committer** (and not per repository), a developer who commits to 10 GHAS-enabled repositories only consumes **one** license. Furthermore, if a user does not commit for 90 days, their license is freed.
By leveraging **Security Configurations**, **Custom Properties**, and **Organization Automation**, we can automate a phased rollout that targets repositories with high developer overlap, effectively scaling your security coverage at $0 additional cost.
# Step 1: Establish Your Baseline Security Configuration
Security configurations simplify the rollout of GitHub security products at scale by defining collections of security settings that can be applied to groups of repositories.
1. Navigate to your Organization page on GitHub.
2. Go to **Settings** > **Code security and analysis** > **Security configurations**.
3. Click **New configuration**.
4. Name it GHAS Standard Enforcement.
5. Enable the following features:
   * **Dependabot alerts** & **Dependabot security updates**
   * **Secret scanning** & **Push protection**
   * **Code scanning** (Default setup)
6. **CRITICAL:** Do *not* apply this to "All repositories" or "Newly created repositories." We want granular, automated control based on custom properties. Click **Save configuration**.
7. Look at the URL in your browser and note the configuration ID at the end (e.g., .../configurations/12345). You will need this for the automation script.

# Step 2: Create Custom Repository Properties
We will use Custom Properties to tag repositories based on their approved rollout phase.
1. In your Organization **Settings**, navigate to **Custom properties** (under the Repository section).
2. Click **Add property**.
3. **Property Name:** ghas_rollout_phase
4. **Property Type:** Single select
5. **Allowed Values:** * pilot (Initial test repositories)
   * phase-1-overlap (Repositories with high committer overlap with the pilot)
   * opt-in (Repositories that request GHAS and are budget-approved)
   * disabled (Repositories out of scope)
6. Click **Save**.

# Step 3: Identify Zero-Cost Repositories
Before enabling GHAS on new phases, identify repositories that won't increase your bill.
1. Run the open-source [ghas-license-utilization](https://github.com/advanced-security/ghas-license-utilization) script (available on GitHub).
2. Use the output to identify repositories where all active committers *already* have a GHAS license from your pilot group.
3. Update the custom property of these highly overlapping repositories to phase-1-overlap.

# Step 4: Automate the Rollout (Org-Wide)
To automate enablement across the organization, we will use an Organization Webhook that passes data to a centralized GitHub Action using the workflow_dispatch REST API.
### 4A: Create the Central Workflow
1. Create a centralized management repository in your organization (e.g., org-management-automation).
2. Create a new file at .github/workflows/ghas-auto-enabler.yml. This workflow accepts the target repository name as an input:
```
name: Enforce GHAS via Custom Properties

on:
  workflow_dispatch:
    inputs:
      target_repository:
        description: 'The name of the repository to evaluate'
        required: true
        type: string

jobs:
  check-and-enable:
    runs-on: ubuntu-latest
    steps:
      - name: Fetch Repository Properties
        id: get_props
        env:
          GH_TOKEN: ${{ secrets.ORG_ADMIN_TOKEN }} 
          TARGET_REPO: ${{ inputs.target_repository }}
        run: |
          # Fetch custom properties for the target repository
          PHASE=$(gh api /repos/${{ github.repository_owner }}/$TARGET_REPO/properties/values --jq '.[] | select(.property_name=="ghas_rollout_phase") | .value')
          echo "phase=$PHASE" >> $GITHUB_OUTPUT

      - name: Apply Security Configuration
        # Only attach the config if the phase matches our budget-approved tags
        if: steps.get_props.outputs.phase == 'pilot' || steps.get_props.outputs.phase == 'phase-1-overlap'
        env:
          GH_TOKEN: ${{ secrets.ORG_ADMIN_TOKEN }}
          CONFIG_ID: '12345' # Replace with your Configuration ID from Step 1
          TARGET_REPO: ${{ inputs.target_repository }}
        run: |
          # Attach the configuration to the repository via the GitHub API
          curl -X PUT \
            -H "Accept: application/vnd.github+json" \
            -H "Authorization: Bearer $GH_TOKEN" \
            -H "X-GitHub-Api-Version: 2022-11-28" \
            [https://api.github.com/orgs/$](https://api.github.com/orgs/$){{ github.repository_owner }}/code-security/configurations/$CONFIG_ID/attach \
            -d '{"repositories":[{"name":"'$TARGET_REPO'"}]}'
```
### 4B: Set up the Webhook Bridge
Because GitHub API endpoints require an Authorization: Bearer <token> header, you cannot point a native GitHub Webhook directly to the API (webhooks only send signatures, not tokens).
To bridge this, configure a lightweight proxy (e.g., AWS Lambda, Azure Function, or a tool like Zapier):
1. Create an **Organization Webhook** listening to Repository events (created/edited).
2. Point the webhook payload to your Proxy URL.
3. Have your Proxy extract the repository.name from the webhook payload.
4. Have your Proxy execute the following standard GitHub REST API call to trigger your automation:

```
curl -L \
  -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer <YOUR_ORG_ADMIN_PAT>" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  [https://api.github.com/repos/YOUR_ORG/org-management-automation/actions/workflows/ghas-auto-enabler.yml/dispatches](https://api.github.com/repos/YOUR_ORG/org-management-automation/actions/workflows/ghas-auto-enabler.yml/dispatches) \
  -d '{"ref":"main","inputs":{"target_repository":"extracted-repo-name"}}'
```
*(Note: Alternatively, creating an internal GitHub App is the official enterprise best practice to listen for org-wide events and make API calls without managing a separate proxy server).*
# Step 5: Monitor and Measure
Once the automation is running, you can track your success.
* Use the **Security overview** at the Enterprise or Organization level to view overall risk and enablement coverage.
* Monitor your **Active Committer count** in the Enterprise billing settings to ensure you are staying comfortably within your licensed limits while maximizing your repository coverage.
