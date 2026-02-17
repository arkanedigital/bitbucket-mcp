# Bitbucket MCP (Arkane Digital Fork)

A Model Context Protocol (MCP) server for Bitbucket Cloud, forked from [MatanYemini/bitbucket-mcp](https://github.com/MatanYemini/bitbucket-mcp).

This fork adds HTTP/Streamable HTTP transport, per-session OAuth token pass-through, and a significantly expanded tool set including branch, file, tag, and commit operations. It is designed for use with [Raikoo](https://raikoo.com) as the MCP client.

## What This Fork Adds

- **HTTP/Streamable HTTP transport** -- the upstream only supports stdio
- **OAuth auto-discovery endpoints** (RFC 8414 and RFC 9728) pointing to Bitbucket's OAuth provider
- **Per-session OAuth token pass-through** -- each MCP session carries its own Bitbucket OAuth token; no server-side credentials required
- **Bearer token authentication** for HTTP transport (`MCP_AUTH_TOKEN`)
- **Branch operations** -- `listBranches`, `getBranch`, `createBranch`, `deleteBranch`
- **File/directory operations** -- `getFileContent`, `listDirectory`, `writeFile`, `writeFiles`, `deleteFile`
- **Tag operations** -- `listTags`, `getTag`, `createTag`, `deleteTag`
- **Commit browsing and diff operations** -- `listCommits`, `getCommit`, `getFileHistory`, `getDiff`, `getDiffStat`, `compareBranches`, `listCommitComments`, `getCommitStatuses`
- **Returns 405 for unsupported GET /sse** per the MCP Streamable HTTP spec

## How to Run

### Quick Start (HTTP Transport)

```bash
cd bitbucket-mcp
npm install
TRANSPORT_MODE=http PORT=11001 npx tsx src/index.ts
```

Or build first:

```bash
npm run build
TRANSPORT_MODE=http PORT=11001 node dist/index.js
```

### Environment Variables

| Variable | Description | Default |
|---|---|---|
| `TRANSPORT_MODE` | `http` for HTTP transport, `stdio` for stdio | `stdio` |
| `PORT` | Port to listen on (be careful not to conflict with other services) | `3000` |
| `HOST` | Bind address | `127.0.0.1` |
| `MCP_AUTH_TOKEN` | Optional static bearer token for server-level authentication (separate from Bitbucket OAuth) | -- |
| `BITBUCKET_TOKEN` | Fallback Bitbucket access token (not needed with per-session OAuth) | -- |
| `BITBUCKET_USERNAME` | Fallback Bitbucket username (not needed with per-session OAuth) | -- |
| `BITBUCKET_PASSWORD` | Fallback Bitbucket app password (not needed with per-session OAuth) | -- |
| `BITBUCKET_URL` | Bitbucket API base URL | `https://api.bitbucket.org/2.0` |
| `BITBUCKET_WORKSPACE` | Default workspace name | -- |
| `BITBUCKET_ENABLE_DANGEROUS` | Set to `true` to enable dangerous tools (e.g., deletions) | `false` |

When using per-session OAuth (the normal Raikoo flow), the server does **not** need `BITBUCKET_TOKEN` or `BITBUCKET_USERNAME`/`BITBUCKET_PASSWORD` -- each request carries its own Bitbucket OAuth bearer token. The fallback variables are only needed if you want a global default credential.

## Bitbucket OAuth Setup

Create an OAuth consumer in your Bitbucket workspace settings:

1. Go to `https://bitbucket.org/{workspace}/workspace/settings/api`
2. Click **Add consumer**
3. Fill in:
   - **Name**: e.g., "Raikoo MCP Integration"
   - **Callback URL**: Your Raikoo OAuth callback:
     - Local: `http://localhost:3000/organization/{organizationId}/oauth/callback`
     - Production: `https://your-domain.com/organization/{organizationId}/oauth/callback`
   - **Check "This is a private consumer"** (gives you a proper client secret)
   - **Permissions**: Select what you need:
     - Account: Read (recommended)
     - Repositories: Read, Write (required for file operations)
     - Pull requests: Read, Write
     - Wikis: Read, Write (if needed)
     - Pipelines: Read (if needed)
4. Save and copy the **Key** (client ID) and **Secret**

### Important Notes on Bitbucket OAuth

- Bitbucket does **not** use OAuth scopes in the authorization URL -- permissions are controlled entirely by the consumer registration.
- Do **not** add `offline_access` scope -- Bitbucket does not recognize it and will reject the authorization. Bitbucket issues refresh tokens by default.
- Do **not** add `openid` scope -- Bitbucket does not support OpenID Connect.
- **Leave the scope field empty** in Raikoo's MCP Tool Wizard.

## Raikoo Setup

1. In Raikoo, navigate to the MCP Tool Wizard (Project > Tools > Add Tool > MCP)
2. Enter the server URL: `http://localhost:11001/mcp` (or your deployed URL)
3. Select transport type: **HTTP**
4. No custom headers needed for Bitbucket
5. The wizard will auto-discover OAuth endpoints from the server
6. Enter your Bitbucket OAuth Key (client ID) and Secret
7. **Leave the scope field empty** -- Bitbucket controls permissions via the consumer, not scopes
8. Click "Save & Authorize" to start the OAuth flow
9. Accept the permissions in Bitbucket's consent screen
10. After authorization, proceed to tool discovery to select which Bitbucket tools to enable

## Available Tools

### Repository Operations

| Tool | Description |
|---|---|
| `listRepositories` | List repositories in a workspace (with optional name filter) |
| `getRepository` | Get details for a specific repository |
| `getRepositoryBranchingModel` | Get the branching model for a repository |
| `getRepositoryBranchingModelSettings` | Get branching model settings |
| `updateRepositoryBranchingModelSettings` | Update branching model settings |
| `getEffectiveRepositoryBranchingModel` | Get the effective branching model (with inherited settings) |
| `getProjectBranchingModel` | Get the branching model for a project |
| `getProjectBranchingModelSettings` | Get project branching model settings |
| `updateProjectBranchingModelSettings` | Update project branching model settings |
| `getEffectiveDefaultReviewers` | Get default reviewers for a repository |

### Pull Request Operations

| Tool | Description |
|---|---|
| `getPullRequests` | List pull requests for a repository (filterable by state) |
| `getPullRequest` | Get details for a specific pull request |
| `createPullRequest` | Create a new pull request |
| `updatePullRequest` | Update an existing pull request |
| `getPullRequestActivity` | Get the activity log for a pull request |
| `approvePullRequest` | Approve a pull request |
| `unapprovePullRequest` | Remove approval from a pull request |
| `declinePullRequest` | Decline a pull request |
| `mergePullRequest` | Merge a pull request (supports merge-commit, squash, fast-forward) |
| `createDraftPullRequest` | Create a draft pull request |
| `publishDraftPullRequest` | Publish a draft pull request for review |
| `convertTodraft` | Convert a regular pull request to draft |
| `getPendingReviewPRs` | List pull requests pending your review |
| `getPullRequestCommits` | List commits on a pull request |
| `getPullRequestStatuses` | List commit statuses for a pull request |

### Pull Request Comments

| Tool | Description |
|---|---|
| `getPullRequestComments` | List comments on a pull request |
| `getPullRequestComment` | Get a specific comment |
| `addPullRequestComment` | Add a comment (general or inline on specific lines) |
| `addPendingPullRequestComment` | Add a pending (batched) comment |
| `publishPendingComments` | Publish all pending comments |
| `updatePullRequestComment` | Update a comment |
| `deletePullRequestComment` | Delete a comment |
| `resolveComment` | Resolve a comment thread |
| `reopenComment` | Reopen a resolved comment thread |

### Pull Request Diffs

| Tool | Description |
|---|---|
| `getPullRequestDiff` | Get the diff for a pull request |
| `getPullRequestDiffStat` | Get diff statistics |
| `getPullRequestPatch` | Get the patch for a pull request |

### Pull Request Tasks

| Tool | Description |
|---|---|
| `getPullRequestTasks` | List tasks on a pull request |
| `createPullRequestTask` | Create a task |
| `getPullRequestTask` | Get a specific task |
| `updatePullRequestTask` | Update a task |
| `deletePullRequestTask` | Delete a task |

### Branch Operations

| Tool | Description |
|---|---|
| `listBranches` | List branches in a repository |
| `getBranch` | Get details for a specific branch |
| `createBranch` | Create a new branch from a source (branch name or commit hash) |
| `deleteBranch` | Delete a branch |

### File/Directory Operations

| Tool | Description |
|---|---|
| `getFileContent` | Get the content of a file at a given path and revision |
| `listDirectory` | List files and directories at a given path |
| `writeFile` | Write (create or update) a single file via commit |
| `writeFiles` | Write multiple files in a single commit |
| `deleteFile` | Delete a file via commit |

### Tag Operations

| Tool | Description |
|---|---|
| `listTags` | List tags in a repository |
| `getTag` | Get details for a specific tag |
| `createTag` | Create a new tag on a target commit |
| `deleteTag` | Delete a tag |

### Commit Operations

| Tool | Description |
|---|---|
| `listCommits` | List commits on a branch or across a repository |
| `getCommit` | Get details for a specific commit |
| `getFileHistory` | Get the commit history for a specific file |
| `getDiff` | Get the diff for a specific commit |
| `getDiffStat` | Get diff statistics for a specific commit |
| `compareBranches` | Compare two branches (diff between refs) |
| `listCommitComments` | List comments on a commit |
| `getCommitStatuses` | Get build/deployment statuses for a commit |

### Pipeline Operations

| Tool | Description |
|---|---|
| `listPipelineRuns` | List pipeline runs (filterable by status, branch, trigger) |
| `getPipelineRun` | Get details for a specific pipeline run |
| `runPipeline` | Trigger a new pipeline run |
| `stopPipeline` | Stop a running pipeline |
| `getPipelineSteps` | List steps for a pipeline run |
| `getPipelineStep` | Get details for a specific step |
| `getPipelineStepLogs` | Get logs for a specific step |

### Pagination

Listing tools accept the following optional parameters:

- `pagelen` -- Number of items per page (default 10, max 100)
- `page` -- 1-based page number
- `all` -- When `true`, follows all pages up to a safety cap of 1,000 entries

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## Links

- **Upstream**: [MatanYemini/bitbucket-mcp](https://github.com/MatanYemini/bitbucket-mcp)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Bitbucket REST API Documentation](https://developer.atlassian.com/cloud/bitbucket/rest/intro/)
