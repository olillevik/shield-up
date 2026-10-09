# Can the agent be stopped from approving PRs?

Research for [#5](https://github.com/olillevik/shield-up/issues/5), part of the map [#1](https://github.com/olillevik/shield-up/issues/1). Checked on 2026-10-09 against docs.github.com, GitHub's REST OpenAPI description, the public GraphQL schema, the source of gh v2.102.0 and github-mcp-server v2.0.2, the Copilot CLI docs and changelog up to 1.0.95, and the Claude Code docs. Nothing was tried against a live repo.

## Short answer

Not with a GitHub permission. Fine-grained PATs and GitHub Apps have a single "Pull requests" repository permission, and its write level covers every review endpoint. Whether a review approves, comments or requests changes is a field in the request body, so any credential that can request changes can also approve. I found no repository role or rule that stops a user from approving. The approval switches GitHub has are for its own bots, GitHub Actions and Copilot code review.

So the block has to act on the requests. GitHub's REST and GraphQL descriptions contain four calls that can record an approving review: two REST calls and two GraphQL mutations, all on `api.github.com`. `gh pr review --approve` sends one of the GraphQL mutations. In all four the approval is a value in the JSON body, and the method and path are the same as for a review that requests changes. A rule that sees only host, method and path can't let change requests through and stop approvals.

Two channels reach GitHub without `gh` or git. Copilot CLI has a built-in GitHub MCP server whose review tool can approve once its write tools are turned on, and the docs don't say where that server runs or which token it uses. Claude Code, when logged in with a claude.ai subscription, loads the user's claude.ai connectors, which can include GitHub, and their traffic goes to `mcp-proxy.anthropic.com`.

## Findings

### No permission separates approval from other reviews

The fine-grained PAT reference lists every review write endpoint under the "Pull requests" permission at write level: create, update, delete pending, dismiss and submit ([fine-grained PAT permissions][fgpat]). None of these rows needs an additional permission. The GitHub App reference has the same rows for both user access tokens and installation access tokens ([GitHub App permissions][app-perms]). Neither reference has a permission or access level for reviews alone. In the REST reference, `event` is a body parameter of "Create a review for a pull request" and "Submit a review for a pull request", with the values `APPROVE`, `REQUEST_CHANGES` and `COMMENT` ([REST reviews][rest-create]).

Both references cover REST only. For GraphQL, GitHub tells App developers to test their app to find out which permissions a query or mutation needs ([choosing permissions][app-choose-gql]), so the permission behind the GraphQL review mutations isn't documented.

A GitHub App user access token can do only what both the app and the user can do ([choosing permissions][app-choose]). That doesn't help here, because the user can approve.

Approval isn't a separate role permission either. Anyone with read access can review a pull request ([about reviews][about-reviews]). The permissions for custom repository roles include "Request a pull request review" and nothing about approving ([custom roles][custom-roles]). Required-review rules decide which approvals count, for example those from people with write permission or from code owners ([protected branches][protected]). They don't stop anyone from submitting an approval.

GitHub's switches for approvals are for its own bots. The repository setting "Allow GitHub Actions to create and approve pull requests" controls whether `GITHUB_TOKEN` can create and approve pull requests ([Actions settings][actions-approve]), and GitHub added an approval policy for Actions in 2022 so that Actions couldn't be used to meet the required-approvals rule ([changelog][actions-changelog]). Copilot approvals are a separate setting, covered under "Whose approval counts" below. The docs name no equivalent for PATs or GitHub App tokens.

### Every call that can record an approval

I searched the REST OpenAPI description ([api.github.com.json][openapi]) for operations whose request body accepts `APPROVE`, and the public GraphQL schema ([schema][gql-schema]) for input fields of type `PullRequestReviewEvent`. Each search found two. These four are every approving call the two descriptions contain:

| Interface | Request | Approves when |
| --- | --- | --- |
| REST | `POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews` | body field `event` is `APPROVE` |
| REST | `POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews/{review_id}/events` | body field `event` is `APPROVE` |
| GraphQL | `POST /graphql`, mutation `addPullRequestReview` | input field `event` is `APPROVE` |
| GraphQL | `POST /graphql`, mutation `submitPullRequestReview` | input field `event` is `APPROVE` |

The REST paths are on `https://api.github.com`, and the GraphQL endpoint for GitHub.com is `https://api.github.com/graphql` ([forming calls][gql-endpoint]). The references for the four calls are [Create a review][rest-create], [Submit a review][rest-submit], [`addPullRequestReview`][gql-add] with [its input][gql-add-input], and [`submitPullRequestReview`][gql-submit] with [its input][gql-submit-input].

The same two REST paths and the same two mutations also create comment and request-changes reviews. A REST rule on method and path alone would block change requests along with approvals, since a change request is a review whose `event` is `REQUEST_CHANGES` ([REST reviews][rest-create]). Single line comments stay possible through `POST /repos/{owner}/{repo}/pulls/{pull_number}/comments` ([review comments][rest-comment]), whose body has no `event` field ([api.github.com.json][openapi]).

For GraphQL, every query and mutation is a `POST` to the same endpoint ([forming calls][gql-post]), so a rule on the path alone blocks every GraphQL call along with the approvals. A finer rule would have to read the GraphQL document and its variables. The GraphQL spec lets a client write the event inline as an enum literal ([enum values][spec-enum]) or pass it in a variable ([variables][spec-vars]), and put several mutations in one request ([mutation][spec-mutation]). An alias renames only the response key ([field alias][spec-alias]), so the field name `addPullRequestReview` or `submitPullRequestReview` is in the document either way. Whether OpenShell can match on any of this is the subject of [#8](https://github.com/olillevik/shield-up/issues/8).

Two REST paths contain "approve" and have nothing to do with pull request reviews. `POST /repos/{owner}/{repo}/issues/{issue_number}/suggestions/{suggestion_id}/approve` approves an issue suggestion, and its description says it "only supports issues, not pull requests". `POST /repos/{owner}/{repo}/actions/runs/{run_id}/approve` approves a workflow run from a public fork ([api.github.com.json][openapi]). The GraphQL mutations `approveDeployments` and `approveVerifiableDomain` approve deployments and a notification domain ([schema][gql-schema]).

### What `gh pr review --approve` sends

In gh v2.102.0, `--approve` sets the review type to `ReviewApprove` ([review.go L102-L104][gh-flag]) and the command calls `api.AddReview` ([review.go L181][gh-call]). `AddReview` turns that type into the GraphQL enum value `APPROVE` and runs the mutation `addPullRequestReview` under the operation name `PullRequestReviewAdd`, with `pullRequestId`, `event` and `body` in the `input` variable ([queries_pr_review.go L249-L274][gh-mutation], [githubv4 enum][githubv4-enum]). On GitHub.com the endpoint is `https://api.github.com/graphql` ([host.go L46-L57][gh-endpoint]), and the GraphQL library gh uses sends the query and variables as one JSON body in a `POST` ([graphql.go L58-L78][gh-post]).

So the request is a `POST` to `https://api.github.com/graphql` whose JSON body holds the mutation text and a `variables.input` object with `event` set to `APPROVE`. A GitHub code search of the cli/cli default branch finds no other use of the approve enum value or of `submitPullRequestReview`. `gh api` takes a REST path or `graphql` as its endpoint ([gh api manual][gh-api]), so it can send any of the four calls directly.

### Pending reviews and later approval

A review created without an event stays pending. The REST reference says so for "Create a review" ([REST reviews][rest-create]), and in GraphQL `event` is optional on `addPullRequestReview` ([input][gql-add-input]). `addPullRequestReviewThread` adds a thread to a pending review ([GraphQL][gql-thread]). In the API, only the two submit calls in the table turn a pending review into an approval. GraphQL `submitPullRequestReview` accepts either the review ID or a pull request ID, the latter "to submit any pending reviews" ([input][gql-submit-input]). The agent acts as the user, so these calls can also submit a pending review that the user started in the browser.

A submitted review can't be turned into an approval later. The REST update call takes `body` as its only body parameter ([update a review][rest-update]), and GraphQL `updatePullRequestReview` "Updates the body of a pull request review" ([GraphQL][gql-update]). No page says in so many words that a submitted review's state is fixed; that follows from the parameters. Dismissing a review sets its state to `DISMISSED` ([review states][gql-states]) and approves nothing.

### Whose approval counts

Pull request authors can't approve their own pull requests ([approving][approving]). If the agent opens pull requests with a repo credential that acts as the user, the user is the author of those pull requests and the agent can't approve them. The exposure is pull requests that someone else opened.

An approval in the user's name counts toward required reviews when the user has write permission ([protected branches][protected]). One rule narrows that. "Require approval of the most recent reviewable push" needs an approval from someone other than the person who pushed last ([protected branches][protected], [rulesets][rulesets]). If the agent pushed last as the user, an approval in the user's name doesn't meet that rule. It applies only where the repository's rules turn it on, and only to pull requests where the agent pushed last.

If the repo credential were a GitHub App installation token, its reviews would come from the app's own identity. I found no GitHub doc that says whether an app's approval counts toward required approvals. GitHub documents this for two bots only. For Actions, the 2022 changelog describes an organization option, on by default at the time, that lets Actions reviews count toward required approvals ([changelog][actions-changelog]). Copilot's reviews don't count toward required approvals by default, and Copilot can submit a counting approval when Copilot approvals are enabled in repository, organization and enterprise settings ([Copilot code review][copilot-approvals]).

### Approvals the agent can cause without approving

One route is GitHub Actions. Changing files in `.github/workflows` takes the "Workflows" permission ([choosing permissions][app-choose-git]), and a `push` runs workflows even when they aren't merged into the default branch ([push event][push-event]). Anyone with write access can set the `GITHUB_TOKEN` permissions in a workflow file ([Actions settings][actions-token]). Where "Allow GitHub Actions to create and approve pull requests" is on, such a workflow can approve a pull request as GitHub Actions ([Actions settings][actions-approve]). The same docs say new repositories in a personal account start with that setting off, and repositories in an organization inherit it from the organization.

The other route is Copilot code review. Where Copilot approvals are enabled, Copilot "can submit an approving review that satisfies your repository's required-approval rule" ([Copilot code review][copilot-approvals]). I didn't confirm which API call requests a Copilot review.

Both depend on settings in the target repo or its organization, and the repo credential has no admin permission to change them.

### Channels other than gh and git

#### Copilot CLI

Copilot CLI has a built-in MCP server, `github-mcp-server`, for "GitHub API integration" ([CLI reference][copilot-builtin]). The CLI reference lists its default tools, and none of them writes reviews. `label_write` is the only write tool in that list. The github-mcp-server install guide says its server "comes pre-installed in Copilot CLI, with read-only tools enabled by default" ([install guide][mcp-copilot]), which disagrees with the CLI reference about `label_write`.

The flags `--add-github-mcp-tool`, `--add-github-mcp-toolset` and `--enable-all-github-mcp-tools` widen the tool set ([CLI reference][copilot-flags]). The changelog for 0.0.388 says the last one "now enables read-write GitHub MCP tools", and the changelog for 1.0.71 says the toolset choice persists in `settings.json` under names such as `githubMcpToolsets` ([changelog][copilot-changelog]). The settings tables in the configuration reference don't list those names ([config reference][copilot-config]).

The server's `pull_request_review_write` tool accepts the `event` values `APPROVE`, `REQUEST_CHANGES` and `COMMENT`. Its `create` method calls `addPullRequestReview`, and its `submit_pending` method calls `submitPullRequestReview` ([pullrequests.go][mcp-review]). The granular tools `create_pull_request_review` and `submit_pending_pull_request_review` cover the same ground ([pullrequests_granular.go][mcp-granular]). So the server uses the two GraphQL mutations from the table.

The docs don't say where the built-in server sends those mutations from. GitHub's hosted MCP server is at `https://api.githubcopilot.com/mcp/` and runs in "GitHub server infrastructure" ([remote server][mcp-remote]). If the built-in server is the hosted one, its GitHub API calls start on GitHub's side, and a rule on `api.github.com` inside the sandbox never sees them.

The docs also don't say which token the server uses. They say only that it needs GitHub authentication ([authentication][copilot-auth]). Copilot CLI takes its GitHub token from `COPILOT_GITHUB_TOKEN`, then `GH_TOKEN`, then `GITHUB_TOKEN`, then its stored OAuth token, then `gh auth token`. The same page warns: "If you set `GH_TOKEN` for another tool, the CLI uses that token instead of the OAuth token from `copilot login`" ([authentication][copilot-auth]). It doesn't list the scopes of a `copilot login` OAuth token, and the changelog for 1.0.88 mentions "GitHub MCP scope escalation" ([changelog][copilot-changelog]).

#### Claude Code

Claude Code has no built-in GitHub tool. Its tools reference lists none, and WebFetch takes a URL and a prompt with no documented way to set a method or a body ([tools reference][claude-tools]). Inside the sandbox it reaches GitHub through Bash, which means `gh`, git or any other program that can read the credential, and through MCP servers.

A user or project can add GitHub's hosted MCP server with a PAT ([MCP][claude-mcp-github]). When Claude Code is logged in with a claude.ai subscription, connectors added in claude.ai load automatically ([MCP][claude-mcp-connectors]). GitHub is one of those connectors ([self-hosted deploy][claude-connector-github]), and connector traffic goes through `mcp-proxy.anthropic.com` ([network config][claude-network]). A connector is authorized in claude.ai, so it acts with that authorization and not with the repo credential. I didn't check which tools the claude.ai GitHub connector has.

Claude Code also has commands that start cloud sessions, such as `/autofix-pr` and `/code-review ultra`, and `/web-setup` connects GitHub for cloud sessions "using your local `gh` CLI credentials" ([commands][claude-commands]). Those sessions run outside the sandbox. Copilot CLI's `/delegate` likewise hands work to Copilot cloud agent, "which runs on GitHub's servers" ([authentication][copilot-auth]).

## Open questions

1. Does an approving review made with a GitHub App installation token count toward required approvals? No GitHub doc I found says, and the answer matters for "Which mechanism mints the repo credential?".
2. Does Copilot CLI's built-in GitHub MCP server run locally or at `api.githubcopilot.com/mcp/`, and which token does it use? This decides whether a rule on `api.github.com` sees its calls.
3. Which OAuth scopes does a `copilot login` token carry, and what does "GitHub MCP scope escalation" add to them?
4. Which permission do the GraphQL review mutations need with a fine-grained PAT? The REST tables say "Pull requests" write, and GitHub doesn't document GraphQL permissions.
5. Which API call requests a Copilot review, and can the claude.ai GitHub connector approve a pull request?
6. Can an OpenShell network policy match a JSON body field or a GraphQL mutation name? That's [#8](https://github.com/olillevik/shield-up/issues/8).

[fgpat]: https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens#repository-permissions-for-pull-requests
[app-perms]: https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps#repository-permissions-for-pull-requests
[app-choose]: https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app#about-github-app-permissions
[app-choose-gql]: https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app#choosing-permissions-for-graphql-api-access
[app-choose-git]: https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app#choosing-permissions-for-git-access
[rest-create]: https://docs.github.com/en/rest/pulls/reviews#create-a-review-for-a-pull-request
[rest-submit]: https://docs.github.com/en/rest/pulls/reviews#submit-a-review-for-a-pull-request
[rest-update]: https://docs.github.com/en/rest/pulls/reviews#update-a-review-for-a-pull-request
[rest-comment]: https://docs.github.com/en/rest/pulls/comments#create-a-review-comment-for-a-pull-request
[openapi]: https://github.com/github/rest-api-description/blob/355db395991cb7e421fae2cc720189ad86558cbe/descriptions/api.github.com/api.github.com.json
[gql-schema]: https://docs.github.com/public/fpt/schema.docs.graphql
[gql-endpoint]: https://docs.github.com/en/graphql/guides/forming-calls-with-graphql#the-graphql-endpoint
[gql-post]: https://docs.github.com/en/graphql/guides/forming-calls-with-graphql#communicating-with-graphql
[gql-add]: https://docs.github.com/en/graphql/reference/pulls#mutation-addpullrequestreview
[gql-add-input]: https://docs.github.com/en/graphql/reference/pulls#input-object-addpullrequestreviewinput
[gql-submit]: https://docs.github.com/en/graphql/reference/pulls#mutation-submitpullrequestreview
[gql-submit-input]: https://docs.github.com/en/graphql/reference/pulls#input-object-submitpullrequestreviewinput
[gql-update]: https://docs.github.com/en/graphql/reference/pulls#mutation-updatepullrequestreview
[gql-thread]: https://docs.github.com/en/graphql/reference/pulls#mutation-addpullrequestreviewthread
[gql-states]: https://docs.github.com/en/graphql/reference/pulls#enum-pullrequestreviewstate
[spec-enum]: https://spec.graphql.org/October2021/#sec-Enum-Value
[spec-vars]: https://spec.graphql.org/October2021/#sec-Language.Variables
[spec-mutation]: https://spec.graphql.org/October2021/#sec-Mutation
[spec-alias]: https://spec.graphql.org/October2021/#sec-Field-Alias
[gh-flag]: https://github.com/cli/cli/blob/v2.102.0/pkg/cmd/pr/review/review.go#L102-L104
[gh-call]: https://github.com/cli/cli/blob/v2.102.0/pkg/cmd/pr/review/review.go#L181
[gh-mutation]: https://github.com/cli/cli/blob/v2.102.0/api/queries_pr_review.go#L249-L274
[gh-endpoint]: https://github.com/cli/cli/blob/v2.102.0/internal/ghinstance/host.go#L46-L57
[gh-post]: https://github.com/cli/shurcooL-graphql/blob/v0.0.4/graphql.go#L58-L78
[githubv4-enum]: https://github.com/shurcooL/githubv4/blob/48295856cce7/enum.go#L1510
[gh-api]: https://cli.github.com/manual/gh_api
[about-reviews]: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/about-pull-request-reviews
[custom-roles]: https://docs.github.com/en/organizations/managing-user-access-to-your-organizations-repositories/managing-repository-roles/about-custom-repository-roles
[protected]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#require-pull-request-reviews-before-merging
[rulesets]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#require-a-pull-request-before-merging
[approving]: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/approving-a-pull-request-with-required-reviews
[actions-approve]: https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#preventing-github-actions-from-creating-or-approving-pull-requests
[actions-token]: https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#setting-the-permissions-of-the-github_token-for-your-repository
[actions-changelog]: https://github.blog/changelog/2022-01-14-github-actions-prevent-github-actions-from-approving-pull-requests/
[push-event]: https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#push
[copilot-approvals]: https://docs.github.com/en/copilot/concepts/agents/code-review#copilot-approvals
[copilot-builtin]: https://docs.github.com/en/copilot/reference/cli-command-reference#built-in-mcp-servers
[copilot-flags]: https://docs.github.com/en/copilot/reference/cli-command-reference#command-line-options
[copilot-config]: https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-config-dir-reference#configuration-file-settings
[copilot-auth]: https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/authenticate-copilot-cli
[copilot-changelog]: https://github.com/github/copilot-cli/blob/dcdc0273c0250d8b32677ca99bc0e1f5006daa89/changelog.md
[mcp-copilot]: https://github.com/github/github-mcp-server/blob/v2.0.2/docs/installation-guides/install-copilot-cli.md
[mcp-remote]: https://github.com/github/github-mcp-server/blob/v2.0.2/docs/remote-server.md
[mcp-review]: https://github.com/github/github-mcp-server/blob/v2.0.2/pkg/github/pullrequests.go#L2005-L2234
[mcp-granular]: https://github.com/github/github-mcp-server/blob/v2.0.2/pkg/github/pullrequests_granular.go
[claude-tools]: https://code.claude.com/docs/en/tools-reference#webfetch-tool-behavior
[claude-mcp-github]: https://code.claude.com/docs/en/mcp#example-connect-to-github-for-code-reviews
[claude-mcp-connectors]: https://code.claude.com/docs/en/mcp#use-mcp-servers-from-claude-ai
[claude-connector-github]: https://code.claude.com/docs/en/self-hosted-environments-deploy#connector-traffic-leaves-your-network
[claude-network]: https://code.claude.com/docs/en/network-config
[claude-commands]: https://code.claude.com/docs/en/commands
