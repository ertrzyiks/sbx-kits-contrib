# Bitbucket API

A mixin kit that wires up Bitbucket authentication for REST API and git HTTPS operations using **Repository Access Tokens**. It handles token authentication through the sandbox proxy, so the token is stored on the host and never lands inside the sandbox. Pairs with any base agent (claude, codex, gemini, …).

Bitbucket is not one of sbx's built-in auto-sign-on services, so out of the box agents inside a sandbox have no credentials for Bitbucket API or git operations. This kit closes that gap: it declares a `bitbucket` credential and injects it into outbound requests at the proxy. The container only ever sees a proxy-managed placeholder in `BITBUCKET_TOKEN`.

## Usage

Store your Bitbucket personal access token (PAT) once on the host. The token needs `repositories:read`, `repositories:write`, and `workspace:read` scopes (or custom scopes depending on your needs):

```console
sbx secret set bitbucket
```

Then create a sandbox with the kit:

```console
sbx run --kit "git+https://github.com/docker/sbx-kits-contrib.git#dir=bitbucket-api" claude
```

Verify inside the sandbox:

```console
# Test with curl
curl -H "Authorization: Bearer $BITBUCKET_TOKEN" \
  https://api.bitbucket.org/2.0/user
```

## How auth works

### REST API Authentication (Bearer)

- The kit declares a `bitbucket` credential with `proxyManaged: true`. Inside the container, `BITBUCKET_TOKEN` is set to a sentinel value, never the real token.
- On any request to `api.bitbucket.org`, the sandbox proxy replaces the `Authorization` header with `Bearer <your-real-PAT>`. The real token never enters the sandbox filesystem or environment.

### Git HTTPS Authentication (HTTP Basic)

- Git HTTPS operations (e.g., `git clone https://bitbucket.org/...`) are authenticated automatically through the proxy.
- The proxy intercepts HTTPS requests to `bitbucket.org` and injects HTTP Basic authentication:
  ```
  Authorization: Basic <base64(x-token-auth:BITBUCKET_TOKEN)>
  ```
- The username `x-token-auth` is Bitbucket's standard identifier for personal access token authentication over HTTPS.
- No additional setup is needed — the proxy handles auth transparently.

## Example usage

### REST API calls

Once authenticated, you can use the Bitbucket REST API v2.0:

```bash
# Get current user info
curl -H "Authorization: Bearer $BITBUCKET_TOKEN" \
  https://api.bitbucket.org/2.0/user

# List repositories in a workspace
curl -H "Authorization: Bearer $BITBUCKET_TOKEN" \
  https://api.bitbucket.org/2.0/repositories/{workspace}

# Get a specific repository
curl -H "Authorization: Bearer $BITBUCKET_TOKEN" \
  https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}

# List pull requests in a repository
curl -H "Authorization: Bearer $BITBUCKET_TOKEN" \
  https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests

# List issues in a repository
curl -H "Authorization: Bearer $BITBUCKET_TOKEN" \
  https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/issues
```

### Git HTTPS operations

Once you've set up your personal access token with `sbx secret set bitbucket`, you can work with repositories using git:

```bash
# Clone a repository (proxy injects token automatically)
git clone https://bitbucket.org/{workspace}/{repo_slug}.git
cd {repo_slug}

# Create and switch to a new branch
git checkout -b feature/my-feature

# Make changes and commit
git add .
git commit -m "Add my feature"

# Push to Bitbucket (proxy injects token automatically)
git push -u origin feature/my-feature

# Pull latest changes
git pull origin main

# Create a pull request via REST API
curl -X POST \
  -H "Authorization: Bearer $BITBUCKET_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "My feature",
    "source": {"branch": {"name": "feature/my-feature"}},
    "destination": {"branch": {"name": "main"}}
  }' \
  https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests
```

Replace `{workspace}` and `{repo_slug}` with your actual Bitbucket workspace and repository names.

## Setting up credentials

This kit requires a **Repository Access Token** (not an App password). Repository Access Tokens are specifically designed for Bearer and x-token-auth authentication and work with both the REST API and git HTTPS operations.

### Create a Repository Access Token

1. Visit your Bitbucket workspace: https://bitbucket.org/{workspace}
2. Navigate to **Settings > Repository access tokens** (under the workspace or project settings)
3. Click **Create token**
4. Give it a label (e.g., "sbx-agent")
5. Select the required scopes:
   - `repository:read` — read repository data
   - `repository:write` — create/update repositories
   - `workspace:membership:read` — read workspace membership (for API access)
6. Choose the expiration date (or set it to never expire)
7. Copy the generated token and store it on the host:
   ```bash
   sbx secret set bitbucket
   ```
   Then paste your token when prompted.

### Why Repository Access Tokens (not App passwords)?

- **App passwords** authenticate via HTTP Basic using your Bitbucket username and are not compatible with Bearer tokens — this kit requires Bearer tokens for the REST API and x-token-auth for git HTTPS.
- **Repository Access Tokens** support both Bearer authentication (for REST API v2.0) and x-token-auth HTTP Basic (for git HTTPS), which is what this kit's proxy configuration uses.

## Cleanup

```console
sbx secret rm -g --service bitbucket
```

## Notes

- **Public repositories**: Public repositories can be cloned without authentication using HTTPS.
- **Token scope**: Ensure your personal access token has the necessary scopes (`repositories:read`, `repositories:write`) for the operations you need.
- **Credential setup required**: Git HTTPS and REST API operations only work if the `bitbucket` credential is properly set up via `sbx secret set bitbucket`.

## Rate limiting

Bitbucket API has rate limits. For details, see the [Bitbucket REST API documentation on rate limiting](https://developer.atlassian.com/cloud/bitbucket/rest/intro/#rate-limiting).
