# Can a repo credential bypass rulesets or branch protection?

Research for #4 (`credential-rule-bypass`) on the shield-up v1 map (#1). The sources were read on 2026-10-09: GitHub's docs, the REST and GraphQL references, the GitHub changelog and the source of `gh` v2.102.0. Each claim links to the page that makes it.

## Short answer

Being a repository admin does not by itself bypass a ruleset. Only the actors on a ruleset's bypass list can bypass it, and GitHub's role table says the admin's right to push to protected branches "doesn't apply to rulesets as these have a different bypass model" ([repository roles][repo-roles]). Classic branch protection works the other way round. Its restrictions do not apply to people with admin permissions unless the rule has "Do not allow bypassing the above settings" turned on ([about protected branches][apb-bypass]).

The docs never say whether a fine-grained PAT without the Administration permission carries its owner's admin bypass. That leaves both of the ticket's admin cases open: a ruleset whose bypass list names repository admins or the user, and a classic rule that does not enforce its restrictions for admins. The PAT docs say a token has its owner's capabilities and "is further limited by any scopes or permissions granted to the token" ([managing your personal access tokens][pat]). The permission tables list no permission for bypass ([fine-grained PAT permissions][perms-fgpat]). Neither page settles whether bypass is something a permission limits. A test on a scratch repo would settle it, and the steps are under "Open questions".

A GitHub App installation token acts as the app, and its requests are attributed to the app ([authenticating as an installation][app-install-token]). It bypasses a ruleset only when the app is on that ruleset's bypass list. Off the list it is held to the rules like any other actor. A GitHub App user access token acts as the user ([authenticating on behalf of a user][app-user-token]), so it has the same open question as the PAT.

The only setup the docs cover without an open question is an installation token from an app that holds no Administration permission and is on no bypass list. For classic rules it rests on one inference, explained under "Which settings hold every push and merge to the rules". For a PAT or a user access token, the docs only guarantee that every push and merge is held to every rule when the target repo holds admins to its rules: no ruleset bypass entry covers the user, and every classic rule enforces its restrictions for admins. The repo credential cannot read either of those settings. None of the three credentials can loosen the rules, because creating, editing or deleting a ruleset or a branch protection rule takes the Administration permission ([fine-grained PAT permissions][perms-fgpat], [GitHub App permissions][perms-app]).

## Rulesets

A branch ruleset's bypass list can hold repository admins, organization owners and enterprise owners, the maintain or write role and custom roles based on write, teams, GitHub Apps, and Dependabot ([creating rulesets][creating-rulesets-bypass]). The REST API names the actor types `Integration`, `OrganizationAdmin`, `RepositoryRole`, `Team`, `DeployKey` and `User`, and says `OrganizationAdmin` "is not applicable for personal repositories" ([rules REST API][rest-rules-create]). Individual users became possible bypass actors on repository-level rulesets on 2026-05-07 ([changelog][cl-user-bypass]).

An admin who is not on the list is held to the ruleset. Besides the role table quoted above, the force-push rule says that when force pushes are blocked, "organization owners or repository administrators will be unable to change or rename the default branch unless they are authorized to bypass the ruleset" ([available rules][available-rules-force]).

Each bypass entry has a mode: `always`, which is the default, `pull_request` or `exempt` ([rules REST API][rest-rules-create]). In `pull_request` mode "the selected actor is now required to open a pull request", and "the actor can then choose to bypass any branch protections and merge that pull request" ([creating rulesets][creating-rulesets-bypass]). In `exempt` mode "rules will not be run for that actor and a bypass audit entry will not be created" ([rules REST API][rest-rules-create]). So every mode lets the actor merge a pull request that has not met the ruleset's requirements. `pull_request` mode only adds the step of opening one.

Anyone with read access can view a repository's active rulesets ([about rulesets][about-rulesets]). The `bypass_actors` property is only returned to a caller with write access to the ruleset ([rules REST API][rest-rules-get]). What the repo credential can read is `current_user_can_bypass`, which GitHub's REST description defines as "the bypass type of the user making the API request for this ruleset", with the values `always`, `pull_requests_only`, `never` and `exempt` ([REST OpenAPI description][openapi]). The field arrived in June 2023 ([changelog][cl-2023]). It is only returned on the repository-level endpoint ([REST OpenAPI description][openapi]), and listing rulesets needs only the Metadata permission ([fine-grained PAT permissions][perms-fgpat]). The docs do not say whether the field takes a token's missing permissions into account.

## Classic branch protection

"By default, the restrictions of a branch protection rule don't apply to people with admin permissions to the repository or custom roles with the 'bypass branch protections' permission" ([about protected branches][apb]). The setting "Do not allow bypassing the above settings" applies the restrictions to admins and those roles too ([about protected branches][apb-bypass]). Its REST field is `enforce_admins` ([branch protection REST API][rest-branch-protection]). The force-push setting sits under "Rules applied to everyone including administrators", so a classic rule blocks force pushes by admins whether or not the bypass setting is on ([managing a branch protection rule][managing-bpr]).

Classic rules have their own allowance lists, such as "Allow specified actors to bypass required pull requests" and the force-push allowance. "Actors may only be added to bypass lists when the repository belongs to an organization" ([managing a branch protection rule][managing-bpr]). Under "Restrict who can push to matching branches", "people and apps with admin permissions to a repository are always able to push to a protected branch or create a matching branch" ([about protected branches][apb-restrict]). The docs do not say whether "Do not allow bypassing" overrides that sentence.

A personal repository has one admin, its owner, and the owner can "merge a pull request on a protected branch, even if there are no approving reviews" ([permission levels for a personal account repository][personal-repo-perms]).

Reading a branch protection rule takes Administration read on a fine-grained PAT ([fine-grained PAT permissions][perms-fgpat]), which the repo credential does not have. The "Get a branch" endpoint needs only Contents read, and its response schema includes `protection.enforce_admins` ([branches REST API][rest-branches]). The docs do not say whether that field is filled in for a caller without admin rights.

## Merging without waiting for requirements

In the web UI an administrator can tick "Merge without waiting for requirements to be met (bypass branch protections)" ([merging with a merge queue][merge-queue]). The API has three calls that merge a pull request directly, and only the newest one asks the caller to opt in to bypass.

The GraphQL `mergePullRequest` mutation has no bypass argument ([GraphQL pulls reference][gql-pulls]). Neither does the REST endpoint `PUT /repos/{owner}/{repo}/pulls/{pull_number}/merge` ([pulls REST API][rest-pulls-merge]). The GraphQL `PullRequest` object has a `viewerCanMergeAsAdmin` field that "indicates whether the viewer can bypass branch protections and merge the pull request immediately" ([GraphQL pulls reference][gql-pulls]). `gh pr merge --admin` is documented as "use administrator privileges to merge a pull request that does not meet requirements" ([gh manual][gh-manual]). In `gh` v2.102.0 the flag only switches off gh's own blocked-state check and the merge queue, and then sends the same `mergePullRequest` mutation as a normal merge ([merge.go][gh-merge-go], [http.go][gh-http-go]). So over GraphQL the server alone decides whether an unready pull request merges, and a credential the server treats as a bypass actor needs no special call. The REST docs do not say whether the synchronous endpoint behaves the same way.

The asynchronous endpoint, `PUT /repos/{owner}/{repo}/pulls/{pull_number}/merge-async`, became generally available on 2026-10-01 and is now the recommended way to merge through the API ([changelog][cl-async-merge]). Its `bypass_rules` parameter defaults to `false` and means "whether to bypass repository rules that the authenticated actor is permitted to bypass" ([pulls REST API][rest-pulls-async]). Bypass is opt-in per call there, which stops an accidental bypass by an honest client and does nothing against an agent that sets the flag.

Both REST merge endpoints need only Contents write on a fine-grained PAT, the same permission that creates commits and refs through the API ([fine-grained PAT permissions][perms-fgpat]). The repo credential can therefore merge any pull request that meets its requirements, and the rules are the only thing that stops it merging one that does not.

## How each credential is judged

### Fine-grained PAT owned by an admin

A fine-grained PAT is tied to the user who made it. It is limited to resources owned by one user or organization, can be limited further to chosen repositories, and has "the same capabilities to access resources and perform actions on those resources that the owner of the token has", narrowed by its permissions ([managing your personal access tokens][pat]). Pushes and merges made with it are the user's, so the ruleset entries that could match it are the repository admin role, the organization admin entry when the user owns the organization, a `User` entry for the user, and any team or role the user holds. Whether a token without Administration still matches those entries is the open question.

The same page lists a gap that matters for the map: fine-grained PATs cannot yet be used "to contribute to repositories where the user is an outside or repository collaborator" ([managing your personal access tokens][pat]).

### GitHub App installation token

Requests made with an installation token are attributed to the app, and the token can only use the permissions the app was granted ([authenticating as an installation][app-install-token]). GitHub Apps can be put on a ruleset bypass list ([creating rulesets][creating-rulesets-bypass]), and the REST API lists `Integration` among the bypass actor types ([rules REST API][rest-rules-create]). Apps have been bypass actors in organization rulesets since 2023 ([changelog][cl-2023]). On the list in `always` mode, the token's pushes and merges bypass the ruleset. In `pull_request` mode its direct pushes are held and its pull request merges can bypass. In `exempt` mode no rule runs for it and no audit entry is written. Off the list, it is held.

For classic rules the docs give the default exemption to people with admin permissions and to custom roles with the bypass permission, and they mention apps only as "people and apps with admin permissions" under push restrictions ([about protected branches][apb-restrict]). An app without the Administration permission has no admin permissions to be exempted by, so by the docs its installation token gets no classic exemption. In organization repos a classic rule can still name the app on its own allowance lists ([managing a branch protection rule][managing-bpr]).

### GitHub App user access token

A user access token acts on behalf of the user. Its requests are attributed to the user, and the audit log lists the user as the actor with "programmatic_access_type" set to "GitHub App user-to-server token". The app can only reach what both the user and the app have access to ([authenticating on behalf of a user][app-user-token]). It has the same open question as the PAT. The docs also do not say whether an `Integration` bypass entry for the app covers the app's user access tokens.

## Which settings hold every push and merge to the rules

| Credential | Ruleset entry for admins or the user | Ruleset entry for the app | Classic rule, admins exempt | Classic rule, admins enforced |
|---|---|---|---|---|
| Fine-grained PAT of an admin | not documented | held | not documented | held |
| Installation token | held | bypasses | held, by inference | held |
| User access token of an admin | not documented | not documented | not documented | held |

"Held, by inference" means the docs give the classic exemption to people with admin permissions and to custom roles, and an app without the Administration permission is neither. The last column assumes the actor is not on the classic rule's own allowance lists, which exist only in organization repos.

From the docs alone, only the installation token row has no unknowns, and only while the app is on no bypass list. That includes repository, organization and enterprise rulesets, and in organization repos the classic allowance lists too. The PAT and user access token rows depend on the open question. Until it is answered, those two credentials are only known to be held on repos where no ruleset entry covers the user and every classic rule enforces its restrictions for admins.

The target repo's admin can change its bypass lists at any time. For the installation token, and for the other two until the test says otherwise, no setting on the credential alone guarantees the result. If the test below shows that `current_user_can_bypass` reports what the credential can do, shield-up can read it with the repo credential at every launch.

## Gaps that are not bypass

Rulesets are available in public repositories on GitHub Free and GitHub Free for organizations, and in public and private repositories on GitHub Pro, GitHub Team and GitHub Enterprise Cloud ([about rulesets][about-rulesets]). Protected branches are available on the same plans ([about protected branches][apb]). A private repo on a Free plan has neither, so there is no rule to hold the agent to and the repo credential can push to any branch, the default branch included.

A required status check accepts a status from any source unless the rule names an expected GitHub App, and "any person or integration with write permissions to a repository can set the state of any status check in the repository" ([available rules][available-rules-checks]). Setting a commit status takes the Commit statuses permission on a fine-grained PAT ([fine-grained PAT permissions][perms-fgpat]). Creating a check run takes the Checks permission on a GitHub App ([GitHub App permissions][perms-app]). A repo credential that holds either one can mark its own commit as passing a required check without bypassing anything.

The fine-grained PAT table lists Workflows as an additional permission on the contents and refs endpoints ([fine-grained PAT permissions][perms-fgpat]). A credential that can change workflow files can change what a required check runs. Whether a plain `git push` that touches `.github/workflows/` also needs Workflows is not stated on the pages read for this note.

Secret scanning push protection is a separate control from rulesets. The person who made a blocked push can bypass the block through the URL in the push error ([push protection on the command line][pp-cli]), unless delegated bypass puts other contributors' bypasses through review ([delegated bypass][delegated-bypass]). The API for the same step, `POST /repos/{owner}/{repo}/secret-scanning/push-protection-bypasses`, needs Contents write and accepts fine-grained PATs and user access tokens but not installation tokens ([fine-grained PAT permissions][perms-fgpat], [GitHub App permissions][perms-app]). For that endpoint "the authenticated user must be the original author of the committed secret" ([secret scanning REST API][rest-secret-scanning]).

## Open questions

The docs leave these open:

1. Whether a fine-grained PAT without Administration, owned by a repo admin, bypasses a ruleset whose bypass list names the repository admin role, the user, or a team the user is in.
2. Whether that PAT gets the classic admin exemption on a rule without "Do not allow bypassing the above settings".
3. Whether a GitHub App user access token behaves like the PAT in both cases, and whether an `Integration` entry for the app also covers its user access tokens.
4. Whether `current_user_can_bypass` and `viewerCanMergeAsAdmin` report what the token can do or what its owner can do.
5. Whether the synchronous REST merge endpoint merges an unready pull request for a bypass actor without being asked to.
6. Whether "Do not allow bypassing the above settings" overrides "always able to push" for admins under push restrictions.
7. Whether the `enforce_admins` field returned by "Get a branch" is filled in for a caller without admin rights.
8. Whether converting a classic rule that exempts admins produces a ruleset that lists repository admins as bypass actors. The conversion docs only say GitHub "generates one or more rulesets that preserve the original rule's behavior" ([converting branch protections][converting]).

### A test that settles them

Use a public repository owned by the user's personal account, so that rulesets and protected branches work on any plan. Give it a default branch with a few commits and a required status check named `never-reports` that nothing ever reports, so the check stays pending.

The test needs four credentials. The first is a fine-grained PAT owned by the user, limited to the scratch repo, with Contents write, Pull requests write, Metadata read and nothing else. The second and third are an installation token and a user access token from a GitHub App that has the same repository permissions and is installed on the scratch repo only. The fourth is the user's own `gh` login, which is the control that shows whether a setup grants the user bypass at all.

| Setup | Rules on the default branch |
|---|---|
| A | ruleset requiring a pull request with one approval and the `never-reports` check, force pushes blocked, empty bypass list |
| B | setup A plus the repository admin role on the bypass list in `always` mode |
| C | setup A plus the repository admin role in `pull_request` mode |
| D | setup A plus the user as a `User` bypass actor in `always` mode |
| E | setup A plus the app as an `Integration` bypass actor in `always` mode |
| F | ruleset A disabled, and a classic rule with the same requirements with "Do not allow bypassing the above settings" off |
| G | setup F with "Do not allow bypassing the above settings" on |

Under each setup, run these probes with each credential:

1. Read `current_user_can_bypass` from `GET /repos/{owner}/{repo}/rulesets` (setups A to E).
2. Push a new commit straight to the default branch with git.
3. Force-push the default branch back by one commit with git.
4. Merge a feature branch into the default branch with `POST /repos/{owner}/{repo}/merges`.
5. Open a pull request with no approval and the check pending, and read its `viewerCanMergeAsAdmin`.
6. Merge three such pull requests, one with `gh pr merge --admin`, one with `PUT /repos/{owner}/{repo}/pulls/{pull_number}/merge` and one with `merge-async` and `bypass_rules: true`.

After any probe that succeeds, reset the default branch with the control credential. Record each probe as accepted or rejected, and check the repository's rule insights page for bypass entries ([managing rulesets][managing-rulesets-insights]).

By the docs, every probe with every credential should be rejected under setups A and G. Under B and D the control should succeed at every probe, under C at the pull request merges in probe 6 only, and under F at everything except the force push in probe 3. Under E the installation token should succeed at every probe and the control should be rejected. A control result on the git pushes in probes 2 and 3 that differs from this means the setup is wrong, and the other results for that setup mean nothing. The PAT's results under B, C, D and F answer questions 1 and 2, and the user access token's results under B to F answer question 3. Comparing probes 1 and 5 with the push and merge results answers question 4. The REST merge in setups where some credential bypasses answers question 5.

If the PAT and the user access token are rejected under B, C, D and F, admin status does not leak through a token without Administration, and those two credentials are held to the rules in every setup the test covers. The installation token is still held only while the app is on no bypass list.

[about-rulesets]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets
[creating-rulesets-bypass]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository#granting-bypass-permissions-for-your-branch-or-tag-ruleset
[available-rules-force]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#block-force-pushes
[available-rules-checks]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#require-status-checks-to-pass-before-merging
[managing-rulesets-insights]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/managing-rulesets-for-a-repository#viewing-insights-for-rulesets
[converting]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/converting-branch-protections-to-rulesets
[apb]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#about-branch-protection-rules
[apb-bypass]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#do-not-allow-bypassing-the-above-settings
[apb-restrict]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#restrict-who-can-push-to-matching-branches
[managing-bpr]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule#creating-a-branch-protection-rule
[repo-roles]: https://docs.github.com/en/organizations/managing-user-access-to-your-organizations-repositories/managing-repository-roles/repository-roles-for-an-organization
[personal-repo-perms]: https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/repository-access-and-collaboration/permission-levels-for-a-personal-account-repository
[merge-queue]: https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/merging-a-pull-request-with-a-merge-queue
[pat]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
[perms-fgpat]: https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens
[perms-app]: https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps
[app-install-token]: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation
[app-user-token]: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-with-a-github-app-on-behalf-of-a-user
[rest-rules-create]: https://docs.github.com/en/rest/repos/rules#create-a-repository-ruleset
[rest-rules-get]: https://docs.github.com/en/rest/repos/rules#get-a-repository-ruleset
[rest-branch-protection]: https://docs.github.com/en/rest/branches/branch-protection#update-branch-protection
[rest-branches]: https://docs.github.com/en/rest/branches/branches#get-a-branch
[rest-pulls-merge]: https://docs.github.com/en/rest/pulls/pulls#merge-a-pull-request
[rest-pulls-async]: https://docs.github.com/en/rest/pulls/pulls#merge-a-pull-request-asynchronously
[rest-secret-scanning]: https://docs.github.com/en/rest/secret-scanning/secret-scanning#create-a-push-protection-bypass
[gql-pulls]: https://docs.github.com/en/graphql/reference/pulls
[openapi]: https://github.com/github/rest-api-description/blob/main/descriptions/api.github.com/api.github.com.yaml
[pp-cli]: https://docs.github.com/en/code-security/how-tos/secure-your-secrets/work-with-leak-prevention/push-protection-on-the-command-line#bypassing-push-protection
[delegated-bypass]: https://docs.github.com/en/code-security/concepts/secret-security/delegated-bypass
[gh-manual]: https://cli.github.com/manual/gh_pr_merge
[gh-merge-go]: https://github.com/cli/cli/blob/v2.102.0/pkg/cmd/pr/merge/merge.go
[gh-http-go]: https://github.com/cli/cli/blob/v2.102.0/pkg/cmd/pr/merge/http.go
[cl-2023]: https://github.blog/changelog/2023-06-27-repository-rules-public-beta-updates/
[cl-user-bypass]: https://github.blog/changelog/2026-05-07-repository-rulesets-user-bypass-and-branch-renaming/
[cl-async-merge]: https://github.blog/changelog/2026-10-01-github-async-merge-api-generally-available/
