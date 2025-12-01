Add minimum token permissions for all GitHub workflow files to improve security posture according to OpenSSF Scorecard recommendations.

## Context
This addresses the Token-Permissions check from https://github.com/ossf/scorecard/blob/ab2f6e92482462fe66246d9e32f642855a691dc1/docs/checks.md#token-permissions.

## Step 1: Add Root-Level Permissions

Every workflow file MUST have a root-level `permissions:` YAML block.

### Rules for Root-Level Permission YAML Block:
1. **🔍 READ THE FILE FIRST**: Always examine the existing formatting style before making any changes
2. Root-Level permissions MUST be limited to either `read-all` or `contents: read`
3. If the existing root-level permission is `read-all` then leave it unchanged
4. If existing root-level permissions have more than read permissions, then move the root-level permission down to the job level where it's needed
   (follow Step 2 for rules about adding job level permissions)
5. **Standard format**: New root-level permissions should be:
   ```yaml
   permissions:
     contents: read
   ```
6. **Placement**: Insert immediately after the root-level `on:` YAML block (no other root-level YAML blocks in between)
7. **Don't reorder existing root-level YAML blocks** - only add the permissions block
8. **Preserve formatting**: Match the existing blank line style (see detailed rules below)
9. **Do not add any comments** to the root-level permissions YAML block

## Step 2: For Regular Workflow Jobs, apply appropriate Job-Level Permissions

Regular Workflow Jobs are defined here as workflow jobs that DO NOT have a `uses:` node directly under the job node and DO have a `steps:` node.

Check these and add job-specific permissions for any that need more than read permission.

### When to Add Job Permissions for Regular Workflow Jobs
- Steps explicitly using `secrets.GITHUB_TOKEN` or `github.token`
  - If the step calls a script, analyze that script to see what permissions are needed
- Steps that use `actions/github-script` implicitly use `github.token`
  - Analyze the script it is executing to see what permissions are needed
- **Steps that call a script**: Analyze the script to see what permissions are needed

### Job Permission Rules for Regular Workflow Jobs
0. If only read permissions are needed then do not insert a new permissions block.
1. **Placement**: Insert the `permissions:` YAML block at the very top of the job YAML block, or directly under a `needs:` block if one exists

   **Example A - Job without `needs:` block:**
   ```yaml
   jobs:
     my-job:
       permissions:
         contents: write  # required for pushing changes
       name: My Job
       runs-on: ubuntu-latest
       steps:
         - name: Checkout
           uses: actions/checkout@v3
   ```

   **Example B - Job with `needs:` block:**
   ```yaml
   jobs:
     my-job:
       needs: [setup, build]
       permissions:
         contents: write  # required for pushing changes
         pull-requests: write  # required for commenting on PRs
       name: My Job
       runs-on: ubuntu-latest
       steps:
         - name: Checkout
           uses: actions/checkout@v3
   ```

2. **Don't reorder existing YAML blocks**
3. **Only add write permissions**
4. **Preserve existing read permissions** if they are already present
5. **Add a trailing comment for each write permission that you add**, explaining briefly why it's needed, e.g. `# required for assigning reviewers to PRs`. Trailing line comments should only have a single space before the `#`.
6. **DO NOT add any comment** for write permissions which were already present
7. **DO NOT add any additional comment** for write permissions moved down from the root-level permissions block

### Common Permission Patterns:
- `JamesIves/github-pages-deploy-action` → needs `contents: write`
- Writing to repository → needs `contents: write`
- Creating releases → needs `contents: write`
- Posting comments → needs `issues: write` or `pull-requests: write`

### ⚠️ Important Exceptions:
- **Steps which use other tokens such as `OPENTELEMETRYBOT_GITHUB_TOKEN`**: Custom tokens don't need workflow permissions
- **Steps which use `actions/cache/save`**: Doesn't require special permissions
- **Don't add unnecessary permissions**: Only add what's actually needed

## Step 3: For Workflow Jobs that call a local reusable workflow, apply appropriate Job-Level Permissions

After applying Step 2 to all workflows, check each workflow job that calls a local reusable workflow.
These are workflow jobs that have a `uses:` node directly under the job node and have NO `steps:` node.

Read the local reusable workflow file and gather all of the permissions that it requires
(from its root-level permission block workflow AND all job-specific permission blocks).
This may include one or more read permissions in addition to write permissions.

This is the set of permissions required by the job that calls the local reusable workflow.

If the local reusable workflow only requires "contents: read" permissions, then do not add a job-specific permission block. Otherwise apply these rules when adding the job-specific permission block:

Notes that only apply to Jobs that call a local reusable workflow
1. **Placement**: If a job-level `permissions:` block already exists, do not reorder it.
   If one does not already exist and you are going to add one, it should be the first line under the job name node. With one exception, if the job has a `needs:` block then add the `permissions:` block directly after that node.
2. If you add a new permissions block, "contents: read" should be the first permission listed under it.
2. **Don't reorder existing YAML blocks**
3. **Add a trailing comment only to the line with text `permissions:`**: `# required by the reusable workflow`. Trailing line comments should only have a single space before the `#`. Don't add any comments to any other lines in this case.
4. **DO NOT** add any comments to any of the read or write permissions in this case. Only add the above comment above as a trailing comment specifically on the `permissions:` line
