# How OpenShell providers inject the repo credential into git and gh

Research for ticket #6 on the shield-up v1 map (#1). The sources are the local `openshell` 0.1.3 CLI, the NVIDIA/OpenShell repository at tag `v0.1.3` (commit `e1f3c82caa3ed3b65de22889ae7ef32a774878ef`), gh 2.102.0 for how `gh` finds and sends a token, and Git's documentation for how `git` asks for credentials. Each `[Sn]` points to a row in the sources table at the end. The investigation was read-only, and nothing here was run in a sandbox.

## Answer

A process in the sandbox does not hold the real token. The sandbox supervisor sets the provider's environment variable, for example `GH_TOKEN`, to a placeholder such as `openshell:resolve:env:v<revision>_GH_TOKEN` ([S1], [S2]). The egress proxy terminates TLS with a per-sandbox CA, finds the placeholder in the outgoing HTTP request and swaps in the real value. It does this only when network policy admits the request and the request's host, port and path fall inside the credential's binding ([S3], [S4]). In a header, the proxy handles a bare placeholder, a placeholder after a scheme word such as `Bearer` or `token`, and a placeholder inside the decoded `user:password` of an `Authorization: Basic` header ([S5], [S6]).

`gh` needs nothing extra. It reads `GH_TOKEN` or `GITHUB_TOKEN` and sends `Authorization: token <placeholder>`, which the proxy rewrites ([S7], [S8]). `git` reads neither variable and sends a token only when a credential helper supplies one, so shield-up has to configure a helper inside the sandbox ([S9], [S10]). `gh auth git-credential` is such a helper. It answers with username `x-access-token` and the placeholder as password, and the proxy can rewrite a Basic header built from that pair ([S11], [S6]).

The provider type that fits is a GitHub profile such as the repository's example `providers/github.yaml`. The gateway ships no profiles, so shield-up has to import one, and the example denies push and API writes until a policy allows them ([S12], [S13], [S14]). A provider can hold a token with an expiry and can be updated while the sandbox runs. A process that is already running keeps the placeholder it started with, though, and the agent's `git` and `gh` inherit it from the agent. Only gateway-managed refresh gives a placeholder that resolves to each new token as soon as the gateway mints it, and its four strategies cover two OAuth2 flows, Google service accounts and AWS STS ([S15], [S16], [S17]). I found no setting that limits a provider to one sandbox. A provider belongs to a workspace, and any sandbox in that workspace can attach it ([S18], [S19]).

## Findings

### What a process in the sandbox sees

The gateway sends the attached providers' real credential values to the sandbox supervisor, and the supervisor turns each into a placeholder before it builds a workload environment. Resolver material stays in the supervisor; only the placeholder map reaches the workload ([S20], [S21]). For the sandbox entrypoint, the supervisor starts from the image environment, adds the provider placeholders, removes every proxy variable (`HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, `NO_PROXY`, their lowercase forms, `grpc_proxy` and `NODE_USE_ENV_PROXY`) and then sets six CA trust variables ([S22], [S23]). Traffic reaches the proxy by routing: the sandbox runs in its own Linux network namespace, and all of its traffic goes through the veth address where the proxy listens ([S24]).

The six trust variables are `NODE_EXTRA_CA_CERTS`, `DENO_CERT`, `SSL_CERT_FILE`, `REQUESTS_CA_BUNDLE`, `CURL_CA_BUNDLE` and `GIT_SSL_CAINFO`. A code comment says `GIT_SSL_CAINFO` is there because git on Ubuntu Noble links libcurl-gnutls, which ignores `SSL_CERT_FILE` ([S25], [S4]).

The provider part of the environment holds one variable per credential key stored in the provider record ([S26]). The only provider plugins that add variables in 0.1.3 are the two Google ones, so a GitHub provider created with `--credential GH_TOKEN` gives the sandbox `GH_TOKEN` and no `GITHUB_TOKEN` ([S27], [S26]). `--from-existing` stores every non-empty variable in the profile's list under its own name, so a host with both `GITHUB_TOKEN` and `GH_TOKEN` set produces both keys ([S28]). The docs describe discovery as storing "the first non-empty local environment value", which reads differently from the code ([S29]).

A static credential's placeholder is `openshell:resolve:env:v<revision>_<KEY>`. A credential under gateway-managed refresh gets `openshell:resolve:env:s<handle>_<KEY>` instead ([S1]). The revision is the first 8 bytes of a SHA-256 digest over the attached provider records and bindings, so it is an opaque fingerprint and not a counter ([S30]). A unit test shows the shape a process sees: `GITHUB_TOKEN=openshell:resolve:env:v10_GITHUB_TOKEN` ([S2]).

### How the proxy replaces the placeholder

Substitution needs the proxy to parse the request as HTTP. TLS is detected and terminated by default. An endpoint set to `tls: skip`, or a non-HTTP tunnel, is relayed as it is and gets no substitution ([S4], [S3]). Two checks must both pass: network policy must allow the calling binary and the destination, and the credential's binding must include the request's host, port and path. A profile's endpoints are the binding by default ([S3], [S31]).

The proxy looks for placeholders in header values, query parameters and URL path segments. Request bodies and WebSocket text frames are included only when an endpoint opts in ([S3]). For a header value it tries three shapes in order. The whole value can be a placeholder. The value can be `Basic <base64>` with a placeholder in the decoded `user:password`, in which case the proxy decodes, replaces and re-encodes. Or the value can be a word, whitespace and a placeholder ([S5], [S6]). The docs show only `Bearer` for the last shape, and the code accepts any word, which covers the `token` scheme that `gh` uses ([S3], [S5]).

A placeholder that is unknown, malformed or expired fails closed and is not forwarded. One sent outside its binding gets HTTP 403 with `credential_endpoint_mismatch` ([S3], [S32]).

The profile fields `auth_style` and `header_name` do not place a static credential. The docs list profile-driven placement on the roadmap and say static injection "still depends on environment placeholders generated from provider credentials" ([S33]). The proxy therefore does not add an `Authorization` header to a request that has none, and the client has to put the placeholder in. The exception is a dynamic `token_grant` credential. The supervisor obtains that token itself and injects it as a header, with nothing in the environment, but it needs a SPIFFE Workload API socket and an OAuth2 token endpoint that accepts a JWT-SVID client assertion ([S34]).

### What git needs

Git reads no token variable. Without a credential helper it tries `GIT_ASKPASS`, `core.askPass`, `SSH_ASKPASS` and then the terminal ([S9]). OpenShell's own end-to-end test for the GitHub provider says that "git only sends the token when a credential helper is configured" ([S10]). I found no OpenShell doc or example at v0.1.3 that configures one, and the default workload image is a minimal Ubuntu without agent tools ([S35]).

A helper that prints the placeholder as the password is enough. `gh auth setup-git --hostname github.com --force` writes the helper `!gh auth git-credential` for `https://github.com` into the global git config, after a blank entry that cuts off any helpers configured earlier ([S36]). `--force` skips the check that gh has a stored login for that host ([S37]). `gh auth git-credential` answers with username `x-access-token` and the active token as password whenever the token's source name ends in `_TOKEN` or gh has no stored user for the host, which covers a sandbox with no gh login ([S11]). Git hands the pair to the server as HTTP credentials ([S9]). If that produces an `Authorization: Basic` header, the proxy replaces the placeholder inside the decoded value ([S6]). The helper reads the variable each time git calls it, so no placeholder is written to `.git/config`.

The example profile names `/usr/bin/git` and `/usr/local/bin/git`, while the HTTPS connection is opened by git's `git-remote-https` helper. It still matches, because policy can authorize a connection through any executable ancestor of the process that owns it ([S38], [S10]). By the same rule, any process that git starts has git as an ancestor and matches the git entry too.

### What gh needs

`gh` reads `GH_TOKEN`, then `GITHUB_TOKEN`, for github.com, and either one takes precedence over stored credentials ([S7]). It sets `Authorization: token <value>` on requests to the token's host and leaves the header off after a redirect to a different host ([S8]). In the sandbox the value is the placeholder, which the proxy replaces for the api.github.com REST and `/graphql` endpoints in the example profile ([S5], [S13]). No `gh auth login` is needed.

gh is a Go program. The `go.mod` of gh 2.102.0 names toolchain go1.27.1, and in that Go release `crypto/x509` on Linux loads its roots from `SSL_CERT_FILE` when the variable is set ([S39], [S40]). The supervisor sets `SSL_CERT_FILE` to the bundle that holds the sandbox CA, so `gh` trusts the proxy without image changes ([S25]).

### Which provider type fits

The gateway contains no built-in profiles, and a provider type is the ID of an imported profile ([S12], [S41]). On the local gateway `openshell profile list` prints "No profiles found." ([S42]). Provider creation rejects a type without a profile, and OpenShell does not activate static credentials from a profileless provider ([S43]).

The repository's example `providers/github.yaml` is the nearest fit. It declares one credential with the variables `GITHUB_TOKEN` and `GH_TOKEN`. It binds that credential to api.github.com, REST and `/graphql`, both read-only, and to github.com for GET, HEAD, OPTIONS and `POST /**/git-upload-pack`. It lets `/usr/bin/gh`, `/usr/local/bin/gh`, `/usr/bin/git` and `/usr/local/bin/git` reach those endpoints ([S13]). Push (`git-receive-pack`) and API writes are denied. The push tutorial grants them with a separate sandbox policy entry, and the push then works with the provider's credential ([S14]). Provider rules and sandbox policy rules are concatenated as separate entries ([S44]).

The map's decision to allow GitHub at host level therefore needs a custom profile with write access, or a base policy entry next to the example profile. The `binaries` paths have to match the image: a profile whose paths match nothing is still listed, but its credential is never injected and its traffic is denied ([S45]). A placeholder sent to a GitHub host outside the profile's endpoints gets `credential_endpoint_mismatch` ([S3]), so every GitHub host where the agent needs the token must be in the profile.

### Short-lived tokens and refresh

A provider credential can carry an expiry. `openshell provider update` takes `--credential-expires-at KEY=TIMESTAMP`, and `provider create` has no such flag ([S42]). The gateway leaves expired credentials out of the sandbox environment, and the proxy refuses an expired value when it resolves a placeholder ([S26], [S32]). `provider update` can replace the token while the sandbox runs, and `--wait` blocks until each attached sandbox has applied it for new processes ([S15], [S42]). The supervisor polls the gateway every 10 seconds by default and reinstalls the environment when the provider revision changes ([S46], [S47]).

The docs say an existing process keeps the revision-scoped placeholder it started with, and only a new process gets the new one ([S15], [S48]). The agent is such a process, and the `git` and `gh` it starts inherit its environment. The code adds detail the docs leave out. The supervisor keeps the resolvers for the last 8 revisions and clears them when a credential's identity changes, for example when a provider is replaced ([S21], [S49]). An old placeholder inside that window resolves to the old token and fails closed once that token's recorded expiry passes. After it drops out of the window it resolves to the current token of the same provider and key ([S50], [S51], [S52]). So with the expiry recorded, an agent session that outlives a one-hour token sees failing `git` and `gh` calls from the old token's expiry until the agent restarts or 8 later revisions push the old one out.

Gateway-managed refresh avoids this. The gateway mints each token, and the workload holds one stable placeholder that resolves to whichever token is current, with no restart ([S16], [S53]). The gateway issues that stable handle only for a credential under a gateway-mintable strategy ([S54]). `provider refresh configure` accepts four: `oauth2-refresh-token`, `oauth2-client-credentials`, `google-service-account-jwt` and `aws-sts-assume-role` ([S42], [S17]). None of them is specific to GitHub. For the OAuth2 strategies the gateway posts a form to the profile's token URL and parses the reply as JSON, and the code sets no `Accept` header ([S55], [S56]). Whether a GitHub token endpoint answers that request in JSON was not checked here.

### Limiting a provider to one sandbox

I found no setting that ties a provider to one sandbox. Providers belong to a workspace, and a sandbox in that workspace attaches one with `--provider` at creation or `openshell sandbox provider attach` later ([S18], [S19]). `provider update --wait` reports one result per attached sandbox, which assumes a provider can be attached to several ([S15]). On a local gateway without OIDC roles every authenticated user is a Platform Admin, so workspaces do not separate local users ([S18]).

Some things are per sandbox. The gateway returns only the providers named in the requesting sandbox's own spec, after checking that the caller is that sandbox ([S20]). A refreshed credential's stable handle is derived from the sandbox ID among other inputs ([S54]). A profile with no endpoints binds its credential only where a sandbox policy endpoint names the provider instance in `credential_binding.provider`, and gateway-global policy does not accept that field ([S31]). Two providers attached to one sandbox cannot expose the same credential variable. Attach, update and refresh configuration all reject that combination ([S57], [S58], [S48]).

### Findings that touch other tickets

For "How do Claude Code and Copilot CLI run and log in inside an OpenShell sandbox?": the example Copilot profile declares `COPILOT_GITHUB_TOKEN`, `GH_TOKEN` and `GITHUB_TOKEN` ([S59]). Since two providers in one sandbox cannot share a variable, the Copilot agent credential and the repo credential need different names, for example `COPILOT_GITHUB_TOKEN` for Copilot and `GH_TOKEN` for the repo. If Copilot CLI then sent the `GH_TOKEN` placeholder to its own API hosts, the proxy would answer `credential_endpoint_mismatch`, because that placeholder is bound to the GitHub profile's hosts ([S3]). Which variable Copilot CLI reads first is open.

For "What can an OpenShell network policy express?": static placeholder resolution checks the endpoint, and binary-scoped credential injection is on the roadmap ([S33]). Any binary that policy lets reach a GitHub host can use the repo credential there, and ancestor matching extends a listed binary to everything it starts ([S38]).

For the two tickets on PR approval: the proxy's L7 rules match REST method and path, and GraphQL operation type, operation name and root fields ([S60]). That is a candidate enforcement point, although the map currently puts path-level GitHub rules out of scope.

For "Which mechanism mints the repo credential?": a credential the gateway can mint through one of its four strategies keeps long agent sessions working. A credential that shield-up refreshes itself through `provider update` fails in any session that outlives one token, at least until the agent restarts, for the reasons in the short-lived tokens section.

## Open questions

1. Nothing here was run in a sandbox, so a hands-on test should confirm `git clone`, `git push` and `gh pr create` through the proxy with a placeholder. The same test should show which HTTP auth scheme GitHub asks git for.
2. Which of `COPILOT_GITHUB_TOKEN`, `GH_TOKEN` and `GITHUB_TOKEN` Copilot CLI reads first.
3. Whether a GitHub token endpoint works with the gateway's `oauth2-refresh-token` or `oauth2-client-credentials` request, which sets no `Accept` header.
4. How shield-up keeps a session alive past a one-hour token if the chosen mechanism needs refreshing from outside the gateway. The sources describe no way to hand a running process a new placeholder.
5. Whether the local macOS gateway provides the SPIFFE Workload API socket that `token_grant` needs, and whether GitHub accepts the bearer header such a grant would inject on git's HTTPS requests.
6. Whether the docs or the code are right about `--from-existing` storing one matching variable or all of them.

## Sources

OpenShell links are pinned to tag `v0.1.3` (commit `e1f3c82caa3ed3b65de22889ae7ef32a774878ef`). gh links are pinned to tag `v2.102.0`.

| ID | Source | What it shows |
|---|---|---|
| S1 | [`secrets.rs` L831-L845](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/secrets.rs#L831-L845) | Placeholder formats `openshell:resolve:env:v<revision>_<KEY>` and `...:s<handle>_<KEY>` |
| S2 | [`provider_credentials.rs` L2047-L2070](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/provider_credentials.rs#L2047-L2070) | Test: a child process sees `GITHUB_TOKEN=openshell:resolve:env:v10_GITHUB_TOKEN` |
| S3 | [Providers overview L364-L420](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx?plain=1#L364-L420) | Placeholder injection, the two checks, supported locations, fail-closed and `credential_endpoint_mismatch` |
| S4 | [Security best practices L116-L127](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx?plain=1#L116-L127) | TLS termination with a per-sandbox CA and the six trust variables |
| S5 | [`secrets.rs` L519-L556](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/secrets.rs#L519-L556) | Header rewrite: bare, Basic and `<word> <placeholder>` shapes |
| S6 | [`secrets.rs` L663-L684](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/secrets.rs#L663-L684) | Basic auth: decode, replace placeholder, re-encode |
| S7 | [`gh help environment` (local gh 2.102.0)](https://cli.github.com/manual/gh_help_environment) | `GH_TOKEN` then `GITHUB_TOKEN`, ahead of stored credentials |
| S8 | [gh `api/http_client.go` L153-L189](https://github.com/cli/cli/blob/v2.102.0/api/http_client.go#L153-L189) | gh sends `Authorization: token <value>` and drops it on a cross-host redirect |
| S9 | [gitcredentials(7)](https://git-scm.com/docs/gitcredentials) | How git asks for HTTP credentials: helpers, then askpass, then the terminal |
| S10 | [`test_sandbox_providers.py` L1070-L1092](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/e2e/python/test_sandbox_providers.py#L1070-L1092) | git sends the token only with a credential helper; `git-remote-https` matches through its `git` ancestor |
| S11 | [gh `gitcredential/helper.go` L113-L141](https://github.com/cli/cli/blob/v2.102.0/pkg/cmd/auth/gitcredential/helper.go#L113-L141) | Helper answers `username=x-access-token` and the active token |
| S12 | [Provider profiles L252](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L252) | The gateway ships no profiles |
| S13 | [`providers/github.yaml` L12-L64](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/providers/github.yaml#L12-L64) | Example GitHub profile: variables, endpoints, rules, binaries |
| S14 | [GitHub push tutorial L43-L211](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/tutorials/github-push-access.mdx?plain=1#L43-L211) | Import, create, attach, denied push, repo-scoped policy that allows push |
| S15 | [Providers overview L176-L202](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx?plain=1#L176-L202) | `provider update --wait`, new processes get the new reference, expiry |
| S16 | [Provider profiles L168-L179](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L168-L179) | Gateway-managed refresh keeps one stable placeholder |
| S17 | [Provider profiles L504-L519](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L504-L519) | Refresh strategies; `configure` accepts only gateway-mintable ones |
| S18 | [Workspaces L11-L64](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/workspaces.mdx?plain=1#L11-L64) | Providers belong to a workspace; local gateways treat users as Platform Admins |
| S19 | [Providers overview L320-L362](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx?plain=1#L320-L362) | Attaching providers at create time and later |
| S20 | [`grpc/policy.rs` L3535-L3613](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L3535-L3613) | Gateway checks sandbox scope and returns the providers in that sandbox's spec |
| S21 | [`provider_credentials.rs` L16-L41](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/provider_credentials.rs#L16-L41) | 8 retained generations; resolver material stays in the supervisor |
| S22 | [`process.rs` L500-L522](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-sandbox/src/process.rs#L500-L522) | Entrypoint environment: image env, provider env, proxy vars stripped, CA vars set |
| S23 | [`process.rs` L125-L176](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-sandbox/src/process.rs#L125-L176) | The stripped proxy variables and the provider env injection |
| S24 | [Security best practices L53-L57](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx?plain=1#L53-L57) | Network namespace routes all traffic to the proxy |
| S25 | [`child_env.rs` L6-L22](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-sandbox/src/child_env.rs#L6-L22) | The six CA trust variables and the `GIT_SSL_CAINFO` comment |
| S26 | [`grpc/provider.rs` L1206-L1266](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/provider.rs#L1206-L1266) | One variable per stored credential key; expired credentials skipped |
| S27 | [`openshell-providers/src/lib.rs` L89-L96](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-providers/src/lib.rs#L89-L96) | Only the Google Cloud and Vertex plugins are registered |
| S28 | [`discovery.rs` L33-L67](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-providers/src/discovery.rs#L33-L67) | `--from-existing` stores every non-empty listed variable |
| S29 | [Provider profiles L487-L491](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L487-L491) | Docs: discovery stores the first non-empty value |
| S30 | [`grpc/policy.rs` L3136-L3187](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L3136-L3187) | Provider environment revision is a truncated SHA-256 digest |
| S31 | [Provider profiles L49-L118](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L49-L118) | Static credential endpoint binding and `credential_binding.provider` |
| S32 | [`secrets.rs` L687-L694](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/secrets.rs#L687-L694) | Expired values do not resolve |
| S33 | [Provider profiles L199-L210](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L199-L210) | Roadmap: `auth_style` placement and binary-scoped injection are not current behaviour |
| S34 | [Provider profiles L558-L654](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L558-L654) | Dynamic token grants: header injection, SPIFFE JWT-SVID, Workload API socket |
| S35 | [Sandboxes overview L241-L247](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/sandboxes/overview.mdx?plain=1#L241-L247) | Default image is a minimal Ubuntu Noble without agent CLIs |
| S36 | [gh `gitcredentials/helper_config.go` L20-L64](https://github.com/cli/cli/blob/v2.102.0/pkg/cmd/auth/shared/gitcredentials/helper_config.go#L20-L64) | `setup-git` writes a blank entry, then `!gh auth git-credential` |
| S37 | [gh `setupgit.go` L86-L99](https://github.com/cli/cli/blob/v2.102.0/pkg/cmd/auth/setupgit/setupgit.go#L86-L99) | `--force` skips the stored-login check for `--hostname` |
| S38 | [Security best practices L80-L92](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx?plain=1#L80-L92) | Binary identity: a connection matches its executable or any ancestor |
| S39 | [gh `go.mod` L3-L5](https://github.com/cli/cli/blob/v2.102.0/go.mod#L3-L5) | gh 2.102.0 uses Go 1.27 with toolchain go1.27.1 |
| S40 | [Go `crypto/x509/root.go` L124-L150 (go1.27.1)](https://github.com/golang/go/blob/go1.27.1/src/crypto/x509/root.go#L124-L150) | `SSL_CERT_FILE` overrides the system roots outside Windows and Apple platforms |
| S41 | [Providers overview L441-L455](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx?plain=1#L441-L455) | Provider types are the IDs of imported profiles |
| S42 | Local `openshell` 0.1.3 CLI | `sandbox create --help` says `--provider` keeps real values from sandbox commands; `provider create --help` has no expiry flag; `provider update --help` has `--credential-expires-at` and `--wait`; `provider refresh configure --help` lists four strategies; `profile list` prints "No profiles found." |
| S43 | [Providers overview L95-L142](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx?plain=1#L95-L142) | Creation rejects profileless types; static credentials need a profile |
| S44 | [Provider profiles L1060-L1082](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L1060-L1082) | Provider and user rules are concatenated; attach and detach |
| S45 | [`providers/README.md` L29-L37](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/providers/README.md?plain=1#L29-L37) | Profiles with non-matching `binaries` never inject their credential |
| S46 | [Supervisor `lib.rs` L1172-L1175](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor/src/lib.rs#L1172-L1175) | Poll interval defaults to 10 seconds |
| S47 | [Supervisor `lib.rs` L4361-L4362](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor/src/lib.rs#L4361-L4362) | Poll loop detects a changed provider environment revision |
| S48 | [Provider profiles L1119-L1131](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L1119-L1131) | Running processes keep their placeholder; env key collisions rejected |
| S49 | [`provider_credentials.rs` L686-L757](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/provider_credentials.rs#L686-L757) | Installing a new revision: generation queue capped, cleared on identity change |
| S50 | [`secrets.rs` L433-L466](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/secrets.rs#L433-L466) | Placeholder lookup and fallback to the current value after ageing out |
| S51 | [`provider_credentials.rs` L2119-L2150](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/provider_credentials.rs#L2119-L2150) | Test: an expired retained generation does not resolve |
| S52 | [`provider_credentials.rs` L2720-L2741](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/provider_credentials.rs#L2720-L2741) | Test: a stale placeholder falls back to the current value after the window |
| S53 | [`provider_credentials.rs` L1695-L1749](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/provider_credentials.rs#L1695-L1749) | Test: a stable handle resolves the refreshed token |
| S54 | [`grpc/provider.rs` L1440-L1496](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/provider.rs#L1440-L1496) | Stable handles only for gateway-mintable refresh, derived from the sandbox ID |
| S55 | [`provider_refresh.rs` L1376-L1400](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/provider_refresh.rs#L1376-L1400) | OAuth2 refresh-token request form |
| S56 | [`provider_refresh.rs` L1622-L1659](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/provider_refresh.rs#L1622-L1659) | Form POST and JSON parse of the token reply |
| S57 | [`grpc/sandbox.rs` L1312-L1418](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/sandbox.rs#L1312-L1418) | Attach validates that env keys stay unique |
| S58 | [`grpc/provider.rs` L2080-L2096](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/provider.rs#L2080-L2096) | Two providers may not provide the same credential env key |
| S59 | [`providers/copilot.yaml` L17-L36](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/providers/copilot.yaml#L17-L36) | Example Copilot profile variables |
| S60 | [Security best practices L98-L103](https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx?plain=1#L98-L103) | L7 rules for REST method and path and GraphQL operations |

[S1]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/secrets.rs#L831-L845
[S2]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/provider_credentials.rs#L2047-L2070
[S3]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx?plain=1#L364-L420
[S4]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx?plain=1#L116-L127
[S5]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/secrets.rs#L519-L556
[S6]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/secrets.rs#L663-L684
[S7]: https://cli.github.com/manual/gh_help_environment
[S8]: https://github.com/cli/cli/blob/v2.102.0/api/http_client.go#L153-L189
[S9]: https://git-scm.com/docs/gitcredentials
[S10]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/e2e/python/test_sandbox_providers.py#L1070-L1092
[S11]: https://github.com/cli/cli/blob/v2.102.0/pkg/cmd/auth/gitcredential/helper.go#L113-L141
[S12]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L252
[S13]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/providers/github.yaml#L12-L64
[S14]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/tutorials/github-push-access.mdx?plain=1#L43-L211
[S15]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx?plain=1#L176-L202
[S16]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L168-L179
[S17]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L504-L519
[S18]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/workspaces.mdx?plain=1#L11-L64
[S19]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx?plain=1#L320-L362
[S20]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L3535-L3613
[S21]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/provider_credentials.rs#L16-L41
[S22]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-sandbox/src/process.rs#L500-L522
[S23]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-sandbox/src/process.rs#L125-L176
[S24]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx?plain=1#L53-L57
[S25]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-sandbox/src/child_env.rs#L6-L22
[S26]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/provider.rs#L1206-L1266
[S27]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-providers/src/lib.rs#L89-L96
[S28]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-providers/src/discovery.rs#L33-L67
[S29]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L487-L491
[S30]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L3136-L3187
[S31]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L49-L118
[S32]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/secrets.rs#L687-L694
[S33]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L199-L210
[S34]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L558-L654
[S35]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/sandboxes/overview.mdx?plain=1#L241-L247
[S36]: https://github.com/cli/cli/blob/v2.102.0/pkg/cmd/auth/shared/gitcredentials/helper_config.go#L20-L64
[S37]: https://github.com/cli/cli/blob/v2.102.0/pkg/cmd/auth/setupgit/setupgit.go#L86-L99
[S38]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx?plain=1#L80-L92
[S39]: https://github.com/cli/cli/blob/v2.102.0/go.mod#L3-L5
[S40]: https://github.com/golang/go/blob/go1.27.1/src/crypto/x509/root.go#L124-L150
[S41]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx?plain=1#L441-L455
[S42]: #sources
[S43]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx?plain=1#L95-L142
[S44]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L1060-L1082
[S45]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/providers/README.md?plain=1#L29-L37
[S46]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor/src/lib.rs#L1172-L1175
[S47]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor/src/lib.rs#L4361-L4362
[S48]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx?plain=1#L1119-L1131
[S49]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/provider_credentials.rs#L686-L757
[S50]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/secrets.rs#L433-L466
[S51]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/provider_credentials.rs#L2119-L2150
[S52]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/provider_credentials.rs#L2720-L2741
[S53]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/provider_credentials.rs#L1695-L1749
[S54]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/provider.rs#L1440-L1496
[S55]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/provider_refresh.rs#L1376-L1400
[S56]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/provider_refresh.rs#L1622-L1659
[S57]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/sandbox.rs#L1312-L1418
[S58]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/provider.rs#L2080-L2096
[S59]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/providers/copilot.yaml#L17-L36
[S60]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx?plain=1#L98-L103
