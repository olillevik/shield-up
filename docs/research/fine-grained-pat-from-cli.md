# Can a fine-grained PAT for one repo be created from a CLI?

Research for [#2](https://github.com/olillevik/shield-up/issues/2), a child of the map [#1](https://github.com/olillevik/shield-up/issues/1). Checked on 2026-10-09 against docs.github.com, the published GitHub REST and GraphQL schemas, the GitHub changelog, and the `gh` manual and source.

## Answer

Not without the browser. GitHub has no API that creates a fine-grained personal access token, so the user clicks through the creation form once per token. shield-up can cut that down to a few clicks with a prefilled URL on `https://github.com/settings/personal-access-tokens/new`. The URL sets the name, description, resource owner, expiry and permissions. No documented parameter selects the repository, so the user still picks the target repo by hand, clicks Generate token and pastes the token into shield-up.

A token with Contents write and Pull requests write, plus Metadata read, can push, open pull requests and review. It needs no Administration permission. The same Pull requests write lets it approve a pull request, and Contents write lets it merge one through the API. No finer permission separates approving from reviewing, or merging from pushing.

Fine-grained tokens do not reach every repo the user can write to. GitHub documents that they cannot be used to contribute where the user is an outside collaborator or a repository collaborator, or to public repos where the user is not a member. On org-owned repos the org can block them, requires owner approval by default, caps their lifetime at 366 days by default and can apply an IP allow list. No API lets the owner list their fine-grained tokens or delete one. The only API that revokes one is the unauthenticated credential revocation endpoint, built for leaked tokens, which takes the token value and emails the owner.

## No API creates a fine-grained PAT

GitHub's published REST descriptions for github.com and for Enterprise Cloud have operations that list, review and revoke fine-grained tokens, and none that creates one ([api.github.com.json](https://github.com/github/rest-api-description/blob/7dee0622aeecf9df3c5060ca28c7a57ee5007804/descriptions/api.github.com/api.github.com.json), [ghec.json](https://github.com/github/rest-api-description/blob/7dee0622aeecf9df3c5060ca28c7a57ee5007804/descriptions/ghec/ghec.json)). I searched every path and operation summary in both files for "token", "credential" and "personal-access". The public GraphQL schema has no mutation that creates or deletes a personal access token ([schema.docs.graphql](https://docs.github.com/public/fpt/schema.docs.graphql)).

The fine-grained token endpoints that do exist sit under `/orgs/{org}/personal-access-tokens` and `/orgs/{org}/personal-access-token-requests`. They serve organization owners, and each one says "Only GitHub Apps can use this endpoint" ([REST API endpoints for personal access tokens](https://docs.github.com/en/rest/orgs/personal-access-tokens)). The org docs add that they "cannot be called with personal access tokens or OAuth apps" ([Reviewing and revoking personal access tokens in your organization](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/reviewing-and-revoking-personal-access-tokens-in-your-organization)).

The documented way to create one is the settings page: Settings, Developer settings, Personal access tokens, Fine-grained tokens, Generate new token ([Managing your personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token)). The same page says "There is a limit of 50 fine-grained personal access tokens you can create" and points automation at GitHub Apps. With one token per target repo, a user could hold tokens for at most 50 repos, fewer if they keep other fine-grained tokens.

`gh` has no command that creates a token. Its `auth` commands are `login`, `logout`, `refresh`, `setup-git`, `status`, `switch` and `token` ([gh auth](https://cli.github.com/manual/gh_auth)). `gh auth login --with-token` reads an existing token from standard input, and the help text says to "Favour setting `GH_TOKEN` for fine-grained personal access token usage" ([gh auth login](https://cli.github.com/manual/gh_auth_login)).

## A prefilled creation URL exists, minus the repository

Since August 2025 the fine-grained creation page reads query parameters, so a link can open the form with fields already filled in ([changelog, 2025-08-26](https://github.blog/changelog/2025-08-26-template-urls-for-fine-grained-pats-and-updated-permissions-ui/)). The docs list these parameters ([Pre-filling fine-grained personal access token details using URL parameters](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#pre-filling-fine-grained-personal-access-token-details-using-url-parameters)):

| Parameter | Valid values | Sets |
|---|---|---|
| `name` | Up to 40 characters, URL-encoded | Display name |
| `description` | Up to 1024 characters, URL-encoded | Description |
| `target_name` | User or organization slug | Resource owner, defaulting to the current user |
| `expires_in` | Integer from 1 to 366, or `none` | Days until expiry, defaulting to 30 or less under a lifetime policy |
| `<permission>` | `read`, `write` or `admin`, as the permission allows | One permission each, such as `contents=write` |

All parameters are optional, and the form validates whether the permissions and resource owner make sense together. The docs' example shows the error `Cannot find the specified resource owner: octodemo` when `target_name` names an org the user is not a member of. Account permissions work only when the user is the resource owner, and organization permissions only when an organization is.

The docs list no parameter for repository access. The changelog says a URL "captures the permissions, resources, lifetime, and details for a token" and names no parameter for "resources" beyond the docs' list. The `gh` project prints one of these URLs in a test script and states the repository scope in a separate line of text for the user to follow ([cli/cli `script/api-host-gateway/run.sh`](https://github.com/cli/cli/blob/ec5b512045db67e5a2a4ff4a1b02660b2fb24390/script/api-host-gateway/run.sh#L50-L57)).

For a target repo `octo-org/octo-repo`, this URL fills in everything the docs allow:

```text
https://github.com/settings/personal-access-tokens/new?name=shield-up+octo-org%2Focto-repo&target_name=octo-org&expires_in=30&contents=write&pull_requests=write
```

The user then chooses "Only select repositories", picks the target repo, clicks Generate token and copies the token into shield-up. The changelog expected template URLs in GHES 3.20.

## Permissions for push, pull requests and review

Each row cites the endpoint's own "Fine-grained access tokens" box or the docs' examples.

| Action | Permission | Source |
|---|---|---|
| Push over HTTPS | Contents write | The docs' "Push access to repositories" template sets `contents=write` ([docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#pre-filling-fine-grained-personal-access-token-details-using-url-parameters)) |
| Open a pull request | Pull requests write | [Create a pull request](https://docs.github.com/en/rest/pulls/pulls#create-a-pull-request) |
| Create or submit a review, `APPROVE` included | Pull requests write | [Create a review](https://docs.github.com/en/rest/pulls/reviews#create-a-review-for-a-pull-request), [Submit a review](https://docs.github.com/en/rest/pulls/reviews#submit-a-review-for-a-pull-request) |
| Comment on the pull request conversation | Issues write or Pull requests write | [Create an issue comment](https://docs.github.com/en/rest/issues/comments#create-an-issue-comment) |
| Push changes to workflow files | Workflows write | The docs' "Update code and open a pull request" template sets `workflows=write` and says it "Includes permission to edit workflow files for Actions" |
| Merge a pull request through the API | Contents write | [Merge a pull request](https://docs.github.com/en/rest/pulls/pulls#merge-a-pull-request) |
| Read repository metadata | Metadata read | The docs' example URL sets only `contents=read` and describes the result as a token with `contents:read` and `metadata:read` |

The review endpoints take an `event` of `APPROVE`, `REQUEST_CHANGES` or `COMMENT` under the same Pull requests write permission ([REST API endpoints for pull request reviews](https://docs.github.com/en/rest/pulls/reviews#create-a-review-for-a-pull-request)). The full permission list has no separate approve permission ([Permissions required for fine-grained personal access tokens](https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens)). A token acts as the user who made it, limited further by its permissions ([About personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#about-personal-access-tokens)). "Pull request authors cannot approve their own pull requests" ([Approving a pull request with required reviews](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/approving-a-pull-request-with-required-reviews)). So an agent holding the token cannot approve pull requests the user opened, its own included, and nothing in the token stops it approving pull requests opened by others.

Contents write also covers `DELETE /repos/{owner}/{repo}/git/refs/{ref}` and `POST /repos/{owner}/{repo}/releases` ([Permissions required for fine-grained personal access tokens](https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens)). None of the actions above needs Administration.

The repository selection leaves public repositories readable. "Tokens always include read-only access to all public repositories on GitHub" ([Managing your personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token)).

Fine-grained tokens cannot call the Checks API ([limitations](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens-limitations)). `gh run view` prints that "it is not currently possible to create a fine-grained PAT with the `checks:read` permission" when it cannot fetch annotations ([cli/cli `pkg/cmd/run/view/view.go`](https://github.com/cli/cli/blob/ec5b512045db67e5a2a4ff4a1b02660b2fb24390/pkg/cmd/run/view/view.go#L409)).

## Organization and enterprise policies

An org owner can block fine-grained tokens from the org's resources, and a blocked org does not appear in the resource owner list on the creation form. Both token types are allowed by default, and public resources stay readable whatever the policy ([Setting a personal access token policy for your organization](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization#restricting-access-by-personal-access-tokens), [Managing your personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token)). An enterprise owner can force restrict or allow for every org in the enterprise, and orgs cannot override it ([Enforcing policies for personal access tokens in your enterprise](https://docs.github.com/en/enterprise-cloud@latest/admin/enforcing-policies/enforcing-policies-for-your-enterprise/enforcing-policies-for-personal-access-tokens-in-your-enterprise)).

Approval is on by default. "Require administrator approval" is the org default ([Setting a personal access token policy for your organization](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization#enforcing-an-approval-policy-for-fine-grained-personal-access-tokens)), and the GA changelog says "developers must request organization owner approval in order to successfully use their fine-grained PAT against their organizations" ([changelog, 2025-03-18](https://github.blog/changelog/2025-03-18-fine-grained-pats-are-now-generally-available/)). Until an owner approves it, the token is `pending` and "will only be able to read public resources". Tokens created by org owners are approved automatically, and the form has an optional justification box for the request ([Managing your personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token)). Owners get a daily email about pending tokens, and the creator gets an email when a token is approved or denied ([Managing requests for personal access tokens in your organization](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/managing-requests-for-personal-access-tokens-in-your-organization)). An enterprise can force approval on or off for all its orgs ([Enforcing policies for personal access tokens in your enterprise](https://docs.github.com/en/enterprise-cloud@latest/admin/enforcing-policies/enforcing-policies-for-your-enterprise/enforcing-policies-for-personal-access-tokens-in-your-enterprise#enforcing-an-approval-policy-for-fine-grained-personal-access-tokens)).

Lifetime is capped at 366 days by default. "For fine-grained personal access tokens, the default the maximum lifetime policy for organizations is set to expire within 366 days" ([Setting a personal access token policy for your organization](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization#enforcing-a-maximum-lifetime-policy-for-personal-access-tokens)). Admins choose a maximum between 1 and 366 days, and the policy applies "when tokens are created, regenerated, or used" ([changelog, 2024-10-18](https://github.blog/changelog/2024-10-18-new-pat-rotation-policies-preview-and-optional-expiration-for-fine-grained-pats/)). The same changelog says a user can create a token with no expiry for personal projects but not against an org they belong to unless an admin relaxes the policy. When an org sets a policy, existing non-compliant tokens are blocked from the org and are not revoked ([Setting a personal access token policy for your organization](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/setting-a-personal-access-token-policy-for-your-organization#enforcing-a-maximum-lifetime-policy-for-personal-access-tokens)). An enterprise can set its own maximum, and orgs in the enterprise can restrict it further ([Enforcing policies for personal access tokens in your enterprise](https://docs.github.com/en/enterprise-cloud@latest/admin/enforcing-policies/enforcing-policies-for-your-enterprise/enforcing-policies-for-personal-access-tokens-in-your-enterprise)). For Enterprise Managed Users, the enterprise policies also apply to the users' own namespaces.

SAML SSO adds no step after creation. "Fine-grained personal access tokens are authorized during token creation, before access to the organization is granted" ([Authorizing a personal access token for use with single sign-on](https://docs.github.com/en/enterprise-cloud@latest/authentication/authenticating-with-single-sign-on/authorizing-a-personal-access-token-for-use-with-single-sign-on)). The docs do not say whether creating the token needs an active SSO session in the browser.

An org IP allow list applies to personal access tokens and to users "with any role or access, including enterprise and organization owners" ([Managing allowed IP addresses for your organization](https://docs.github.com/en/enterprise-cloud@latest/organizations/keeping-your-organization-secure/managing-security-settings-for-your-organization/managing-allowed-ip-addresses-for-your-organization)). Requests from the sandbox would need to leave from an allowed address.

Internal repos in an enterprise must be granted to a fine-grained token, and the token cannot reach internal resources outside the org it targets ([Managing your personal access tokens, Enterprise Cloud](https://docs.github.com/en/enterprise-cloud@latest/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens-limitations), [changelog, 2025-03-18](https://github.blog/changelog/2025-03-18-fine-grained-pats-are-now-generally-available/)).

## Repos a fine-grained PAT cannot reach

The docs list these gaps ([Fine-grained personal access tokens limitations](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens-limitations)):

- contributing to public repos where the user is not a member
- contributing to repos where the user is an outside or repository collaborator
- accessing more than one organization with one token
- accessing Packages
- calling the Checks API
- accessing Projects owned by a user account

The GA changelog lists the same collaborator gap as "Contributing to repositories where you're an outside collaborator or an unaffiliated open source contributor" ([changelog, 2025-03-18](https://github.blog/changelog/2025-03-18-fine-grained-pats-are-now-generally-available/)). I found no later changelog entry that closes it. Repos owned by another user, where the user is a collaborator, and org repos where the user is an outside collaborator are therefore out of reach for a fine-grained token. Among personal access tokens, the docs name classic tokens for these cases: "Outside collaborators can only use personal access tokens (classic) to access organization repositories that they are a collaborator on." A classic token "will grant access to all repositories within the organizations that you have access to, as well as all personal repositories in your personal account" ([Personal access tokens (classic)](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#personal-access-tokens-classic)). A classic token cannot be held to one repo.

## Listing and revoking from the CLI

There is no API for a user to list their own fine-grained tokens. The org endpoints that list tokens with access to an org are callable only by GitHub Apps ([REST API endpoints for personal access tokens](https://docs.github.com/en/rest/orgs/personal-access-tokens)). On Enterprise Cloud, enterprise owners have a token inventory API that covers fine-grained tokens ([enterprise token inventory](https://docs.github.com/en/enterprise-cloud@latest/rest/enterprise-admin/token-inventory)).

A user deletes a token on the settings page; the docs describe no other way for the owner ([Deleting a personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#deleting-a-personal-access-token)). The credential revocation API, `POST /credentials/revoke`, accepts `github_pat_` tokens ([Revoke a list of credentials](https://docs.github.com/en/rest/credentials/revoke#revoke-a-list-of-credentials)). It "is intended to revoke credentials the caller does not own", it returns 403 to any authenticated request, and it is limited to 60 unauthenticated requests per hour and 1000 tokens per request. Revocation is logged in the owner's security log, and the owner gets an email ([changelog, 2026-03-26](https://github.blog/changelog/2026-03-26-credential-revocation-api-now-supports-github-oauth-and-github-app-credentials/)). shield-up could revoke its own token by sending the token value, outside the endpoint's stated purpose and with an email to the user each time.

An org owner can revoke a token's access to the org in the org settings or through the GitHub-App-only API. The token can still read the org's public resources afterwards, and SSH keys it created keep working ([Reviewing and revoking personal access tokens in your organization](https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/reviewing-and-revoking-personal-access-tokens-in-your-organization)).

A token is revoked automatically when it expires, when it is pushed to a public repo or gist, and when it has not been used for a year ([Token expiration and revocation](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/token-expiration-and-revocation)).

## Notes for other tickets

"Can a GitHub App mint per-repo credentials that act as the user?" `POST /applications/{client_id}/token/scoped` turns a user access token into one limited to named repositories and permissions. It needs Basic authentication with the app's `client_id` and `client_secret` ([Create a scoped access token](https://docs.github.com/en/rest/apps/apps#create-a-scoped-access-token)).

"Can the agent be stopped from approving PRs?" and "How does shield-up stop the agent from approving PRs?" With a fine-grained token, Pull requests write includes `APPROVE`. In the pages read for this ticket, the only limit on approving is the rule that authors cannot approve their own pull requests.

"Which repository permissions does the repo credential carry?" Contents write also merges pull requests, deletes refs and creates releases through the API. Workflows write is needed to push workflow files. The Checks API is out of reach.

"Can a repo credential bypass rulesets or branch protection?" A fine-grained token acts as its owner. The pages read for this ticket do not say whether the owner's place on a bypass list carries over to the token.

"What can an OpenShell network policy express?" Every fine-grained token can read all public repositories, so the network policy and the token together do not limit reads to the target repo. Org IP allow lists apply to the sandbox's egress address.

"Which mechanism mints the repo credential?" A fine-grained token fails the "any repo the user can write to" property for collaborator repos, and it needs org approval by default on org repos.

## Open questions

1. Does the creation page accept an undocumented parameter for repository selection? The docs list none, and the changelog's "resources" names no parameter. Opening the URL creates nothing, so a hands-on check on a scratch account is safe.
2. Does creating a fine-grained token for a SAML SSO org need an active SSO session in the browser? The docs say only that the token is authorized during creation.
3. What does the prefilled form do when `target_name` names an org that blocks fine-grained tokens, or when `expires_in` exceeds the org's maximum lifetime? The docs cover only the not-a-member error.
4. How can shield-up tell a pending token from an approved one? I found no endpoint for the token owner, and a pending token can still read public resources.
5. Do `gh pr checks` and the GraphQL status rollup work with a fine-grained token, given the Checks API gap? The `gh` source documents the gap only for run annotations.
