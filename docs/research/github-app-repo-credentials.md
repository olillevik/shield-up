# Can a GitHub App mint per-repo credentials that act as the user?

Research for [#3][issue-3], a child of the map [#1][issue-1]. The sources are GitHub's docs, its REST reference and its changelog, read on 2026-10-09. Each claim links to the page it comes from, and the text says so where the sources are silent or disagree.

## Answer

Yes, through a user access token, which is the only GitHub App token that acts as the user. A CLI can get one through the device flow with nothing but the app's client ID, so shield-up would hold neither a private key nor a client secret. The token request takes a `repository_id` that confines the token to one repository, and requests made with the token are attributed to the user, with the app's badge on their avatar. The token lives 8 hours. Its refresh token lives 6 months and can be redeemed without a client secret, because the token came from the device flow.

An installation access token fits the "one repo, chosen permissions" half better, since both are set at mint time. But it acts as the app's bot account, lives 1 hour, and has to be replaced by minting a new one. Minting it needs the app's private key, and GitHub says an app that runs on a user's device must never ship one.

The user access token has two limits. The device flow cannot narrow permissions per token, so every token carries the app registration's permissions cut down to what the user can do; narrowing them per token needs an endpoint that requires the client secret. And the app must be installed on the account that owns the target repo. On an org repo, a user who is admin of that repo can install the app unless the org has blocked that or the app asks for organization permissions or "Repository administration". A user with only write access cannot install it. GitHub sends the org owner a request instead, and the credential does not work on that repo until an owner installs the app there. A private app installs only on its owner's account, so the app shield-up uses must be public to reach org repos.

## The two token types side by side

The sections below cite the source for each row.

| Property | Installation access token | User access token |
|---|---|---|
| Prefix | `ghs_` | `ghu_`, refresh token `ghr_` |
| One repo at mint time | `repositories` or `repository_ids`, up to 500 | `repository_id`, one repo, ignored if the app or user cannot access it |
| Permissions at mint time | `permissions`, up to what the app was granted | None in the device flow; the scoped-token endpoint needs the client secret |
| Upper bound | The app's permissions and the installation's repos | App permissions and user permissions intersected, in accounts where the app is installed |
| Lifetime | 1 hour | 8 hours, refresh token 6 months |
| Renewal | Mint a new token, which needs a JWT signed with the private key | Single-use refresh token that rotates both tokens |
| Secret needed to mint | Private key, to sign the JWT | Client ID only, in the device flow |
| Acts as | The app's bot account | The user, with the app's badge |

## Installation access tokens

To mint one, the app signs a JWT with its private key, with an expiry no more than 10 minutes ahead ([Generating a JWT][jwt]), and sends it to `POST /app/installations/{installation_id}/access_tokens` ([REST: create an installation access token][rest-iat]). The request body can name up to 500 repositories in `repositories` or `repository_ids` and can set a `permissions` object. The token cannot reach repositories the installation was not granted, or permissions the app was not granted ([Generating an installation access token][iat]).

The token expires after 1 hour. Using an expired token returns 401 and "requires creating a new installation token" ([REST: create an installation access token][rest-iat]). It works for git over HTTPS as the password, with `x-access-token` as the username, if the app has the "Contents" permission ([Authenticating as an installation][as-inst]). It can revoke itself through `DELETE /installation/token` ([REST: revoke an installation access token][rest-inst-revoke]).

Since 27 April 2026 GitHub has been rolling out a stateless format for new installation tokens, a `ghs_`-prefixed JWT of about 520 characters. Code that stores or validates these tokens has to allow for that length ([Changelog 2026-05-15][cl-stateless]).

## User access tokens

GitHub says CLI tools should use the device flow ([Generating a user access token][uat-device]). The app owner has to select "Enable Device Flow" in the app settings first ([Registering a GitHub App][register]). The CLI sends its client ID to `https://github.com/login/device/code`, shows the user a short code such as `WDJB-MJHT` to enter at `https://github.com/login/device`, and polls `POST https://github.com/login/oauth/access_token` with `client_id`, `device_code` and `grant_type` until the user approves. The flow asks for no client secret at any step ([Generating a user access token][uat-device]).

The poll request also takes an optional `repository_id`, which GitHub describes as "The ID of a single repository that the user access token can access. If the GitHub App or user cannot access the repository, this will be ignored." ([Generating a user access token][uat-device]). Read literally, a token requested for a repo where the app is not installed comes back without the one-repo restriction. `GET /user/installations/{installation_id}/repositories` lists the repos a user token can reach in one installation ([REST: list repositories accessible to the user access token][rest-inst-repos]), but the docs do not say whether that list reflects `repository_id`.

The device flow has no permissions parameter. A user access token "only has permissions that both the user and the app have", and it reaches only resources that both can access ([Generating a user access token][uat]). Its permissions are therefore the ones in the app registration, cut down to what the user can do. `POST /applications/{client_id}/token/scoped` can narrow an existing user token to chosen repositories and permissions, but it requires Basic authentication with the client ID and the client secret ([REST: create a scoped access token][rest-scoped]). The docs give no lifetime for a scoped token.

The token expires after 8 hours and its refresh token after 6 months ([Generating a user access token][uat]). App owners can turn expiry off, and GitHub "strongly recommends" leaving it on ([Registering a GitHub App][register]). A refresh is a `POST` to the same token URL with `grant_type=refresh_token`, and the `client_secret` is "Required unless the user access token was generated using the device flow". The response carries a new access token and a new refresh token, and after a refresh "that refresh token and the old user access token will no longer work" ([Refreshing user access tokens][refresh]). The refresh request has no `repository_id` parameter, and the docs do not say whether a refreshed token keeps the restriction.

User access tokens also work as the HTTP password for git, like installation tokens ([Choosing permissions: Git access][perm-git]). In an org that uses SAML SSO, the user needs an active SSO session when they authorize the app, or the token cannot see that org's resources ([SAML and GitHub Apps][saml], [Best practices for creating a GitHub App][best]).

Revoking one user token through `DELETE /applications/{client_id}/token` needs Basic authentication with the client secret ([REST: delete an app token][rest-oauth-delete]). The user can revoke the whole authorization in their account settings, which revokes every token that came from it ([Token expiration and revocation][revocation]). The unauthenticated credential revocation API accepts `ghu_` and `ghr_` tokens. It is meant for exposed credentials, it notifies the credential owner, and it allows 60 requests an hour ([REST: revoke a list of credentials][rest-revoke]).

## Whose name is on the work

Requests made with an installation token "are attributed to the app" ([Authenticating as an installation][as-inst]), and the token "identifies the app as a GitHub App bot account, such as @jenkins\[bot]" ([Differences between GitHub Apps and OAuth apps][diff]). Pull requests, reviews and comments created with it show the bot.

Requests made with a user access token "will be attributed to that user". The UI shows the user's avatar with the app's identicon badge on it, and audit log entries list the user as the actor with `programmatic_access_type` set to "GitHub App user-to-server token" ([Authenticating on behalf of a user][on-behalf]).

Neither token decides who authored a commit. GitHub links a commit to an account by matching the email address in the commit header ([Why are my commits linked to the wrong user?][wrong-user]), and git writes that header from its own configuration where the commit is made. The pages read for this note do not say which account GitHub records as the pusher of a `git push` made with each token type.

GitHub marks a bot-signed commit as verified only when the request "is verified and authenticated as the GitHub App or bot and contains no custom author information, custom committer information, and no custom signature information" ([About commit signature verification][sig-bots]). The docs describe no equivalent for commits made with a user access token.

## Approving pull requests

Approve, request changes and comment are one endpoint, `POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews`, told apart by its `event` field ([REST: create a review][rest-reviews]). All three need the "Pull requests" write permission, for user and installation tokens alike ([Permissions required for GitHub Apps][perm-req]). The permissions on those pages cannot allow comments and change requests without also allowing approval. A user token acts as the user, and "Pull request authors cannot approve their own pull requests" ([Approving a pull request with required reviews][approving]). The agent could therefore not approve a PR the user opened, but it could approve a PR someone else opened. The docs read here do not say whether an approval from an app's bot account counts toward required reviews.

## Holding the private key, or not

The private key "grants access to every account that the app is installed on". For an app that "runs on a user device", GitHub's best practices say "you must never ship your private key with your app", and they go on: "You should not generate installation access tokens since doing so requires a private key. Instead, you should generate user access tokens." ([Best practices for creating a GitHub App][best]). Private keys do not expire, have to be revoked by hand, and an app can hold up to 25 of them ([Managing private keys][keys]).

That leaves shield-up, a local tool with no server, two ways to use a GitHub App.

The first is one shared public app with the device flow turned on. The CLI embeds only the client ID, mints user access tokens and never touches a private key. The app needs no webhook, so its owner can clear "Active" in the webhook settings ([Registering a GitHub App][register]). The cost is trust. Whoever owns the app can generate a private key at any time ([Managing private keys][keys]) and with it reach every installation, and GitHub tells installers to "ensure you trust the owner of the GitHub App" ([Installing a GitHub App from a third party][third-party]). The best-practices page also says public clients "are trivially spoofable - anyone can reuse your app's client ID to sign in", and that "an attacker can use the device flow to remotely impersonate your app as part of a phishing attack", so the page says to enable the device flow only for CLIs, IoT devices and headless systems ([Best practices for creating a GitHub App][best]). If the owner adds permissions later, each installation keeps its old permissions until the account owner approves the new ones ([Choosing permissions][perm]).

The second is an app per user. The user registers it by hand or through the manifest flow, which hands back the private key as a PEM file along with the client secret and webhook secret, and which has to be completed within an hour ([Registering a GitHub App from a manifest][manifest]). The key stays on the user's host outside the sandbox, and shield-up mints installation tokens scoped to the target repo. Those tokens act as the app's bot, and the key sits on a user device, the case in which the best-practices page says to generate user access tokens instead. To reach org repos the app must be public, because "Private GitHub Apps can only be installed on the user or organization account of the app owner" ([Making a GitHub App public or private][visibility]). The manifest flow sends its code back to a `redirect_url`, and its parameter list has no field that turns on the device flow ([Registering a GitHub App from a manifest][manifest]).

## Installing the app on the target repo

A user access token reaches resources only in accounts where the app is installed, and each user who wants tokens has to authorize the app themselves ([Authenticating on behalf of a user][on-behalf]). The person installing picks "All repositories" or "Only select repositories" ([Installing a GitHub App from a third party][third-party]).

Anyone can install an app on their own personal account, and organization owners can install it on their organization. A repository admin can install an app for the repos they administer when the app requests no organization permissions and not the "Repository administration" permission. They can also add their repos to an installation an org owner already made, whatever its permissions. Org members and outside collaborators who cannot install can still pick the org during installation. GitHub then notifies the org owner, who "can modify the repositories that you selected and choose whether to install the GitHub App". The app manager role does not let anyone install ([Installing a GitHub App from a third party][third-party], [Requesting a GitHub App from your organization owner][requesting]).

Org owners can withdraw the right of repository admins to install apps, and those admins then have to use the request flow too ([Limiting app access requests and installations][limiting]). The changelog announced that setting as a public preview on 2025-11-17 ([Changelog 2025-11-17][cl-block-admins]). Owners can also choose who may send requests: members and outside collaborators, which is the default, members only, or nobody. The changelog calls that control generally available from 2026-01-12 ([Changelog 2026-01-12][cl-requests]), while the docs page still says "Blocking app access requests from organization members is in public preview" ([Limiting app access requests and installations][limiting]). The two sources disagree.

The docs describe the request flow for organizations only. For a repo owned by someone else's personal account, they name no way for a collaborator to get the app installed, which leaves it to that account's owner.

## Open questions

- Does a token refreshed from a device-flow token minted with `repository_id` stay confined to that repository? The refresh request has no `repository_id` parameter and the docs are silent.
- How can shield-up confirm that `repository_id` took effect, given that GitHub ignores it without an error when the app or user cannot access the repo? The docs do not say whether `GET /user/installations/{installation_id}/repositories` reflects it.
- Does a user token keep its user's ruleset bypass? Rulesets accept roles, teams and GitHub Apps as bypass actors ([Creating rulesets for a repository][rulesets]), and a user token has the capabilities "that the owner of the token has, and is further limited by any scopes or permissions granted to the token" ([Authenticating on behalf of a user][on-behalf]). Neither page says whether a user who can bypass a ruleset still bypasses it when acting through a user token that lacks the "Administration" permission.
- Does the limit of ten tokens "per user/application/scope combination", after which GitHub revokes older tokens, apply to GitHub App user tokens? The page names OAuth apps only ([Token expiration and revocation][revocation]). It matters if each persistent sandbox holds its own token.
- Which account does GitHub record as the pusher of a `git push` made with each token type, in the UI, in push webhooks and in the audit log?
- Can commits made with a user access token be signed in a way GitHub verifies, when the repo's rules require signed commits?
- Can the manifest flow send its code back to a loopback address, so a CLI could register a per-user app without a web server? The docs do not say.
- Who would own and run a shared shield-up app, given that its owner can mint installation tokens for every repo it is installed on? This is a design question for the mechanism ticket, not a docs question.

[issue-1]: https://github.com/olillevik/shield-up/issues/1
[issue-3]: https://github.com/olillevik/shield-up/issues/3
[approving]: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/approving-a-pull-request-with-required-reviews
[as-inst]: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation
[best]: https://docs.github.com/en/apps/creating-github-apps/about-creating-github-apps/best-practices-for-creating-a-github-app
[cl-block-admins]: https://github.blog/changelog/2025-11-17-block-repository-administrators-from-installing-github-apps-on-their-own-now-in-public-preview/
[cl-requests]: https://github.blog/changelog/2026-01-12-controlling-who-can-request-apps-for-your-organization-is-now-generally-available/
[cl-stateless]: https://github.blog/changelog/2026-05-15-github-app-installation-tokens-per-request-override-header/
[diff]: https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps#token-based-identification
[iat]: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app
[jwt]: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-json-web-token-jwt-for-a-github-app
[keys]: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/managing-private-keys-for-github-apps
[limiting]: https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/limiting-oauth-app-and-github-app-access-requests-and-installations
[manifest]: https://docs.github.com/en/apps/sharing-github-apps/registering-a-github-app-from-a-manifest
[on-behalf]: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-with-a-github-app-on-behalf-of-a-user
[perm]: https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app#about-changes-to-permissions
[perm-git]: https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app#choosing-permissions-for-git-access
[perm-req]: https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps#repository-permissions-for-pull-requests
[refresh]: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/refreshing-user-access-tokens
[register]: https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app
[requesting]: https://docs.github.com/en/apps/using-github-apps/requesting-a-github-app-from-your-organization-owner
[rest-iat]: https://docs.github.com/en/rest/apps/apps#create-an-installation-access-token-for-an-app
[rest-inst-repos]: https://docs.github.com/en/rest/apps/installations#list-repositories-accessible-to-the-user-access-token
[rest-inst-revoke]: https://docs.github.com/en/rest/apps/installations#revoke-an-installation-access-token
[rest-oauth-delete]: https://docs.github.com/en/rest/apps/oauth-applications#delete-an-app-token
[rest-reviews]: https://docs.github.com/en/rest/pulls/reviews#create-a-review-for-a-pull-request
[rest-revoke]: https://docs.github.com/en/rest/credentials/revoke#revoke-a-list-of-credentials
[rest-scoped]: https://docs.github.com/en/rest/apps/apps#create-a-scoped-access-token
[revocation]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/token-expiration-and-revocation
[rulesets]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository#granting-bypass-permissions-for-your-branch-or-tag-ruleset
[saml]: https://docs.github.com/en/apps/using-github-apps/saml-and-github-apps
[sig-bots]: https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification#signature-verification-for-bots
[third-party]: https://docs.github.com/en/apps/using-github-apps/installing-a-github-app-from-a-third-party#requirements-to-install-a-github-app
[uat]: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-user-access-token-for-a-github-app
[uat-device]: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-user-access-token-for-a-github-app#using-the-device-flow-to-generate-a-user-access-token
[visibility]: https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/making-a-github-app-public-or-private
[wrong-user]: https://docs.github.com/en/pull-requests/committing-changes-to-your-project/troubleshooting-commits/why-are-my-commits-linked-to-the-wrong-user
