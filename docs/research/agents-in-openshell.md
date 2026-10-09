# How do Claude Code and Copilot CLI run and log in inside an OpenShell sandbox?

Research for [#7][issue-7], a child of the map [#1][issue-1]. The sources are the local `openshell` 0.1.3 CLI, the `NVIDIA/OpenShell` repo at tag `v0.1.3` (commit `e1f3c82`), Anthropic's Claude Code docs and GitHub's Copilot docs, read on 2026-10-09. OpenShell links are pinned to the tag. Each claim links to the page or file it comes from, and the text says so where the sources are silent or disagree.

## Answer

No image that OpenShell 0.1.3 supports ships `git`, `gh`, Claude Code or Copilot CLI. The default workload image is a minimal Ubuntu 24.04 with 92 packages and none of the four, and the community catalog that used to carry agent images is retired. shield-up has to build its own image. That image needs `git`, `gh` and both agents installed as real executables at fixed paths, with agent auto-update turned off, because OpenShell pins every executable it authorises to the hash it first saw.

The agent credential reaches the agent through a provider. OpenShell puts an opaque placeholder in the agent's environment and swaps in the real value at its egress proxy, on the hosts the provider profile binds and nowhere else. A 0.1.3 gateway has no built-in profiles, so the repo's example `claude-code` and `copilot` profiles have to be copied, corrected and imported. OpenShell discovers local credentials from environment variables only. It cannot reuse a Claude subscription login from the macOS Keychain or a Copilot login from the system keychain.

For Claude Code, an Anthropic Console API key in `ANTHROPIC_API_KEY` is the path OpenShell documents. A Claude subscription has no documented path that keeps the token outside the sandbox. The closest is a `CLAUDE_CODE_OAUTH_TOKEN` from `claude setup-token` in a custom profile, which no source shows working in an interactive session. Running `/login` inside the sandbox works, but it stores the real OAuth token inside the sandbox.

For Copilot CLI, the Copilot-only token that GitHub documents is a fine-grained personal access token owned by the user's personal account, with the "Copilot Requests" account permission and no repository permissions. Like every fine-grained token it can still read all public repositories. Two things are undocumented. One is whether Copilot CLI accepts a placeholder in place of a token, since it checks token prefixes. The other is whether it sends the token to `api.github.com`, which GitHub's allowlist names for Copilot and OpenShell's example profile does not bind. A hands-on test has to settle both.

The minimum hosts are `api.anthropic.com` for Claude Code, and for Copilot CLI the `githubcopilot.com` API hosts and `copilot-proxy.githubusercontent.com`, plus probably two paths on `api.github.com`. The tables under [Network hosts](#network-hosts) list the rest with their purpose.

## The sandbox image

Without `--from`, a sandbox runs the gateway's default workload image, `nvcr.io/nvidia/base/ubuntu:24.04`. The image "does not include agent CLIs or an image-baked OpenShell policy" ([Manage sandboxes][os-sbx-default]), and the 0.1.0 upgrade notes say bare image aliases are gone and users should build an image "with the required agents, tools, and startup behavior" ([Upgrade to 0.1.0][os-upgrade-image]). I read the package list the image carries at `/opt/nvidia/ubuntu-base-image/packages.txt`, pulled through the registry API from the arm64 manifest under index digest `sha256:43c1682f10cdf6561e22f6da1765d491b6f0e5808f94d5fde29d7bdbe75142d5`, build date 2026-09-30. It lists 92 packages. `bash` is there. `git`, `gh`, `curl`, `ca-certificates`, `nodejs` and `libsecret-1-0` are not. The build history shows one `apt-get upgrade` on top of upstream `ubuntu:noble` and no package installs.

The retired `NVIDIA/OpenShell-Community` repo used to publish a `base` image with `git`, `gh`, Node.js 22, Copilot CLI from npm and Claude Code from the native installer, copied to `/usr/local/bin/claude` ([community base Dockerfile][community-dockerfile]). Its README says those images "are no longer maintained or supported" ([OpenShell Community README][community-readme]). That Dockerfile is the layout the example provider profiles assume.

OpenShell identifies a process by "the real path of its executable, as the kernel reports it", so a profile's `binaries` must name real paths ([Network rules: binary matching][os-binary-matching]). Claude Code's native installer makes `~/.local/bin/claude` a symlink into `~/.local/share/claude/versions/` ([Claude Code setup: auto-updates][claude-auto-update]), so the real path changes with each version. Anthropic's apt, dnf and apk packages do not auto-update ([Claude Code setup: update][claude-update]), but the docs do not say where they put the binary ([Install with Linux package managers][claude-pkg]). The example `claude-code` profile expects `/usr/bin/claude` or `/usr/local/bin/claude` ([claude-code.yaml][os-profile-claude]). Copilot CLI's install script installs to `/usr/local` when run as root, and npm needs Node.js 22 ([Installing Copilot CLI][copilot-install]). The example `copilot` profile lists `/usr/bin/copilot` and a path inside the npm global tree, and not `/usr/local/bin/copilot` ([copilot.yaml][os-profile-copilot]).

OpenShell also hashes each executable the first time it takes part in a connection and denies later connections if the file at that path changes ([Network rules: binary matching][os-binary-matching], [Security best practices][os-best-binary]). Both agents update themselves by default. Claude Code's native install updates in the background ([Claude Code setup: auto-updates][claude-auto-update]), and Copilot CLI downloads updates at the start of each session ([Copilot CLI configuration directory][copilot-config]). An update that rewrites the authorised file trips the hash pin, and one that installs to a new path falls outside the profile. The image should pin both versions and set `DISABLE_UPDATES=1` for Claude Code ([Claude Code setup: disable auto-updates][claude-disable-update]) and `COPILOT_AUTO_UPDATE=false` for Copilot CLI ([Copilot CLI environment variables][copilot-env]).

OpenShell handles TLS trust and proxy routing itself. It terminates TLS with a per-sandbox CA and points `NODE_EXTRA_CA_CERTS`, `SSL_CERT_FILE`, `GIT_SSL_CAINFO` and three other variables at that CA, or at a bundle that includes it, for child processes ([Security best practices: TLS][os-best-tls], [child_env.rs][os-code-tls-env]). Claude Code reads `NODE_EXTRA_CA_CERTS` ([Claude Code custom CA certificates][claude-ca]); the Copilot CLI docs I read say nothing about CA variables. OpenShell also strips `HTTP_PROXY`, `HTTPS_PROXY` and `NODE_USE_ENV_PROXY` from child processes and mediates their traffic transparently ([process.rs][os-code-proxy-env]), so neither agent needs proxy settings.

## How a provider carries the agent credential

A provider is a stored credential plus a profile that declares the credential's environment variables, the endpoints it may reach and the binaries allowed to reach them ([Provider profiles][os-profiles-intro]). In the sandbox each credential variable holds a placeholder of the form `openshell:resolve:env:<KEY>`, optionally with a revision or handle before the key ([secrets.rs][os-code-placeholder]). The docs state that "the agent process inside the sandbox never sees real credential values" ([Providers: credential injection][os-prov-inject]). The proxy swaps a placeholder found as a whole header value, after a scheme such as `Bearer`, inside Basic auth, in a URL path or in a query string ([Providers: supported injection locations][os-prov-locations], [secrets.rs header rewrite][os-code-header]). It does so only when network policy allows the binary and destination and the request's host, port and path fall inside the credential's binding, which by default is every endpoint in the profile ([Provider profiles: endpoint binding][os-profiles-check]). Anywhere else the request fails with HTTP 403 `credential_endpoint_mismatch`, and an unresolved placeholder is not forwarded ([Providers: fail-closed behavior][os-prov-failclosed]).

A gateway "lists an empty catalog until you import one" ([Provider profiles: catalog][os-profiles-catalog]), and the local gateway agrees: `openshell profile list` prints `No profiles found.` The files in `providers/` are examples. Their README says to copy one, edit `binaries` and `endpoints` to match the image, and import the copy, because a profile whose binaries do not match the image "matches nothing" ([providers/README.md][os-providers-readme]).

`--auto-providers` applies only when `--provider <name>` names a provider that does not exist yet and matches an imported profile ID. The CLI then runs profile discovery ([Providers: auto-discovery shortcut][os-prov-auto], [provider.rs][os-code-ensure]). Discovery reads environment variables and nothing else; the only discovery context in the code is `std::env::var` ([context.rs][os-code-context]). So OpenShell does not pick up a Claude login from the Keychain or `~/.claude/.credentials.json`, a Copilot login from the `copilot-cli` keychain entry, or the output of `gh auth token`. A maintainer ([MAINTAINERS.md][os-maintainers]) closed the request for Claude subscription support ([#1925][os-1925]) in favour of a planned gateway-owned `provider login` command, and wrote that the new design "does not import or reuse an existing host Claude Code login". That plan ([#3331][os-3331]) is open, and its build plan says the Claude Code adapter "remains dependent on Anthropic approval". The 0.1.3 CLI has no `provider login` command.

Two discovery details affect shield-up. The docs say discovery "stores the first non-empty local environment value" ([Provider profiles: discovery][os-profiles-discovery]), but the code stores every non-empty variable the credential lists ([discovery.rs][os-code-discovery]). The example `copilot` profile lists `COPILOT_GITHUB_TOKEN`, `GH_TOKEN` and `GITHUB_TOKEN` ([copilot.yaml][os-profile-copilot]), so discovery on a host where `GH_TOKEN` is set copies that token into the Copilot provider. The gateway also refuses a sandbox create or a provider attach when two attached providers expose the same credential variable ([sandbox.rs create][os-code-unique-create], [sandbox.rs attach][os-code-unique-attach]), and the example `github` profile uses `GITHUB_TOKEN` and `GH_TOKEN` as well ([github.yaml][os-profile-github]). shield-up should therefore create the Copilot provider itself with the bare form `--credential COPILOT_GITHUB_TOKEN`, which reads the value from the CLI's environment ([Providers: bare key form][os-prov-bare]). The docs warn, for refresh material, that a value expanded into an argument shows up in the host process table ([Providers: credential refresh][os-prov-refresh-secret]).

Two more limits apply. A value passed with `sandbox create --env` is readable by the agent, and the CLI only warns about it ([Manage sandboxes: environment variables][os-sbx-env]). And a process keeps the placeholder it started with, so after a credential update the agent has to be restarted to pick up the new value ([Provider profiles: runtime limitations][os-profiles-runtime]).

## Claude Code

Anthropic documents seven credential sources in a fixed order ([Authentication precedence][claude-precedence]) and where a `/login` is stored ([Credential management][claude-storage]). Four sources matter here.

| Source | What Anthropic says about it | What OpenShell 0.1.3 can do with it |
|---|---|---|
| `ANTHROPIC_AUTH_TOKEN` | Sent as `Authorization: Bearer`, for an LLM gateway or proxy | No example profile declares it |
| `ANTHROPIC_API_KEY` | A Console API key, sent as `X-Api-Key` | The example `claude-code` profile declares it |
| `CLAUDE_CODE_OAUTH_TOKEN` | A one-year subscription token from `claude setup-token` | Needs a custom profile |
| Subscription login from `/login` | Kept in the macOS Keychain, or in `~/.claude/.credentials.json` on Linux | Not discoverable |

With an API key, the example `claude-code` profile reads `ANTHROPIC_API_KEY` or `CLAUDE_API_KEY`, sends it as `x-api-key`, and binds it to `api.anthropic.com`, `statsig.anthropic.com` and `sentry.io` for the `claude` binary ([claude-code.yaml][os-profile-claude]). The provider docs show `--provider claude-code -- claude` finding `ANTHROPIC_API_KEY` and launching Claude Code ([Providers: auto-discovery shortcut][os-prov-auto]). They add that it has to be a Console API key: "Subscription users must generate a separate API key from the Anthropic Console" ([Providers: available provider types][os-prov-note]). In interactive mode Claude Code asks once whether to use the key and remembers the answer ([Authentication precedence][claude-precedence]). With the key approved it skips the browser login ([Log in to Claude Code][claude-login]). The Anthropic pages I read do not mention `CLAUDE_API_KEY`, `statsig.anthropic.com` or `sentry.io`, and they list `api.anthropic.com` and two Datadog hosts for telemetry ([Network access requirements][claude-network]). The example profile and Anthropic's current docs disagree on both counts. OpenShell's end-to-end test of this profile checks that the sandbox sees a placeholder in `ANTHROPIC_API_KEY` and not the secret; it does not start `claude` ([provider_auto_create.rs][os-e2e]). Neither vendor says whether Claude Code checks the shape of a key before sending it.

For a subscription, `claude setup-token` prints a one-year OAuth token for Pro, Max, Team or Enterprise plans that "can only make model requests" and that Claude Code reads from `CLAUDE_CODE_OAUTH_TOKEN` ([Generate a long-lived token][claude-setup-token]). Anthropic suggests the same variable for dev containers and Codespaces ([Development containers][claude-devcontainer]). No example profile declares it. Anthropic does not say which header or hosts receive this token. OpenShell would resolve it as a bare header value or after `Bearer` ([secrets.rs header rewrite][os-code-header]) on any host shield-up's profile binds. One user reported in OpenShell [#620][os-620] in March 2026, through a `generic` provider type that 0.1.3 no longer creates ([Providers: create a provider][os-prov-profileless]), that with the token proxied this way `claude auth status` showed a login while interactive `claude` still opened the browser login. That report is a reason to test and settles nothing.

Running `/login` inside the sandbox works in a container: when the browser cannot reach Claude Code's local callback, the user pastes a code into the terminal ([Log in to Claude Code][claude-login]). On Linux the login then lives in `~/.claude/.credentials.json` ([Credential management][claude-storage]), inside the sandbox, against the map's rule that the real agent credential stays out. Both claude.ai and Console sign-ins need `platform.claude.com` for token exchange and refresh ([Network access requirements][claude-network]).

## Copilot CLI

Copilot CLI takes a GitHub token from `COPILOT_GITHUB_TOKEN`, `GH_TOKEN` or `GITHUB_TOKEN` in that order, then from the `copilot-cli` keychain entry, then from `gh auth token` ([Authenticating Copilot CLI][copilot-auth]). It accepts these token types ([Supported token types][copilot-tokens]):

| Token | Prefix | Copilot CLI support |
|---|---|---|
| OAuth token from `copilot login` | `gho_` | Supported |
| Fine-grained personal access token | `github_pat_` | Supported if owned by the personal account and granted the Copilot Requests account permission |
| GitHub App user-to-server token | `ghu_` | Supported through an environment variable |
| Personal access token (classic) | `ghp_` | Rejected |

A GitHub token that carries only Copilot access is therefore a user-owned fine-grained token with the Copilot Requests permission and repository access set to public repositories ([Authenticating with environment variables][copilot-auth-env]). Fine-grained tokens "always include read-only access to all public repositories on GitHub" ([Managing your personal access tokens][gh-pat]). The docs do not list the scopes of the `gho_` token that `copilot login` stores, so nothing shows that token is Copilot-only.

The scope matters because Copilot CLI ships a built-in GitHub MCP server for issues, pull requests, labels, commits and Actions, and the tools GitHub lists for it include `label_write` ([Built-in MCP servers][copilot-mcp]). `--enable-all-github-mcp-tools` widens the tool set and `--disable-builtin-mcps` turns the server off ([Command-line options][copilot-options]). GitHub lists the GitHub MCP server among the features that need GitHub authentication ([Unauthenticated use][copilot-unauth]) but does not say which credential it uses. In a shield-up sandbox the only GitHub login Copilot CLI has is the agent credential.

The example `copilot` profile sends the token as a bearer header to `api.githubcopilot.com`, its individual, business and enterprise variants, and `copilot-proxy.githubusercontent.com`. Its header says two telemetry hosts are let through without the credential ([copilot.yaml][os-profile-copilot]). Two documented behaviours leave it uncertain that Copilot CLI works with a placeholder. First, Copilot CLI inspects the token: a `ghp_` value is ignored with a warning in an interactive session and stops a `-p` run ([Token (classic) rejected][copilot-classic]). The docs say nothing about a value with no recognised prefix, and a placeholder starts with `openshell:resolve:env:`. Second, GitHub's allowlist names `https://api.github.com/user` and `https://api.github.com/copilot_internal/*` as Copilot "User Management" endpoints ([Copilot allowlist reference][gh-allowlist]). If Copilot CLI sends its GitHub token there, OpenShell answers 403 `credential_endpoint_mismatch`, because the example profile does not bind `api.github.com` ([Providers: fail-closed behavior][os-prov-failclosed]). A shield-up profile could bind the token to those two paths only, since a binding matches host, port and path ([Provider profiles: endpoint binding][os-profiles-check]). The allowlist covers Copilot clients in general, and the Copilot CLI docs publish no host list of their own.

The order Copilot CLI reads variables in helps shield-up. If the repo credential reaches the sandbox as `GH_TOKEN` or `GITHUB_TOKEN`, a set `COPILOT_GITHUB_TOKEN` still wins ([Authenticating Copilot CLI][copilot-auth]). If the Copilot provider is missing, Copilot CLI falls through to the repo credential's placeholder. When the repo credential's profile binds only GitHub's own hosts, that placeholder fails closed on the `githubcopilot.com` hosts.

Running `/login` inside the sandbox uses the device code flow in "known remote or headless environments" ([Authenticating with OAuth][copilot-auth-oauth]). Without a system keychain, and the default image has no `libsecret-1-0`, Copilot CLI asks whether to store the token in plaintext in `~/.copilot/config.json` ([How Copilot CLI stores credentials][copilot-auth-store]). That puts the real token in the sandbox.

## Network hosts

A network rule that lists an agent's binary also covers the processes the agent starts ([Network rules: binary matching][os-binary-matching]), so tools the agent runs can reach its hosts with the placeholder they inherit. The GitHub hosts that `git` and `gh` need belong to the network policy ticket and are left out here.

Claude Code hosts come from [Network access requirements][claude-network]. The last column is my reading of what a shield-up session needs.

| Host | Anthropic's stated purpose | Needed by shield-up |
|---|---|---|
| `api.anthropic.com` | API requests, feature flags, telemetry events, WebFetch domain check | Always |
| `platform.claude.com` | Console sign-in, and OAuth token exchange, refresh and revocation for claude.ai accounts | For a login inside the sandbox; the first-run connectivity check also probes it |
| `claude.ai` | claude.ai account authentication | For a login inside the sandbox |
| `http-intake.logs.us5.datadoghq.com`, `browser-intake-us5-datadoghq.com` | Operational telemetry and error reports | No, with `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` |
| `downloads.claude.ai` | Installer, auto-updater and plugin executable downloads | No, once the image pins the version |
| `github.com`, `raw.githubusercontent.com`, `storage.googleapis.com` | Plugin marketplaces, release notes, plugin metadata | Only for plugins and release notes |
| `mcp-proxy.anthropic.com`, `code.claude.com` | claude.ai connectors, documentation lookups | Optional |

`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` turns off auto-updates, telemetry, error reporting, `/feedback` and release notes ([Environment variables][claude-env]). It does not touch the WebFetch domain check, which calls `api.anthropic.com` unless `skipWebFetchPreflight` is set ([WebFetch domain safety check][claude-webfetch]).

Copilot hosts come from GitHub's [Copilot allowlist reference][gh-allowlist], which covers all Copilot clients and not the CLI alone.

| Host | GitHub's stated purpose | In OpenShell's example profile |
|---|---|---|
| `*.githubcopilot.com`, or the plan-specific `*.individual.`, `*.business.` and `*.enterprise.githubcopilot.com` | API service for Copilot suggestions | The `api.` host and its three plan variants, with the credential |
| `copilot-proxy.githubusercontent.com` | API service for Copilot suggestions | Yes, with the credential |
| `origin-tracker.githubusercontent.com` | API service for Copilot suggestions | No |
| `api.github.com/user`, `api.github.com/copilot_internal/*` | User management | No |
| `github.com/login/*` | Authentication | No |
| `collector.github.com`, `copilot-telemetry.githubusercontent.com` | Analytics and client telemetry | No; the profile has `telemetry.enterprise.githubcopilot.com` |
| `default.exp-tas.com` | Client experimentation | Yes, without the credential |

## Effects on other tickets

The ticket on how providers inject the repo credential into `git` and `gh` inherits the variable clash: the example `github` and `copilot` profiles both declare `GH_TOKEN` and `GITHUB_TOKEN`, and OpenShell refuses to attach two providers that expose the same variable. The example `github` profile also denies `git push` and every API write ([github.yaml][os-profile-github]), so shield-up needs its own GitHub profile.

The tickets on stopping the agent from approving PRs gain a second route to GitHub. Copilot CLI's built-in GitHub MCP server acts with GitHub authentication, so the Copilot agent credential has to carry no repository permissions. Starting Copilot CLI with `--disable-builtin-mcps` is a weaker control, because the agent's shell can start another `copilot` process that inherits the same network rules and placeholder.

The ticket on creating a fine-grained token from a CLI also decides how a user gets a Copilot agent credential, since the documented Copilot-only token is a fine-grained token. The GitHub App ticket touches it too, because Copilot CLI accepts `ghu_` tokens.

The network policy ticket can rely on three OpenShell facts from this research. Rules match binaries by real path. Tools the agent starts inherit its rules. Credential bindings can be narrowed to a path.

## Open questions

- Does Claude Code run interactively with only a placeholder in `ANTHROPIC_API_KEY`, and does its one-time key approval come back when the placeholder changes after a provider update?
- Does Claude Code accept a placeholder in `CLAUDE_CODE_OAUTH_TOKEN`, which header and hosts receive it, and does interactive `claude` skip the browser login with it?
- Does Copilot CLI accept a token value with no recognised prefix, such as an OpenShell placeholder?
- Does Copilot CLI send its GitHub token to `api.github.com`? If it trades the token for a Copilot session token there, that session token is minted inside the sandbox and is a real credential.
- Does Copilot CLI trust the OpenShell CA through `NODE_EXTRA_CA_CERTS` or `SSL_CERT_FILE`? Its docs are silent.
- Which scopes does the `gho_` token from `copilot login` carry?
- Can a GitHub App be given the Copilot Requests permission, so that a `ghu_` token could act as a Copilot-only agent credential?
- Does the sandbox home directory, with Claude Code's settings and Copilot CLI's `~/.copilot`, survive `sandbox stop` and `start`? OpenShell says filesystem persistence "follows the compute driver's stop and start behavior" ([Manage sandboxes][os-sbx-restart]).
- Does Claude Code start when its first-run connectivity check cannot reach `platform.claude.com`?
- Where do Anthropic's apt packages install the `claude` binary?

[issue-1]: https://github.com/olillevik/shield-up/issues/1
[issue-7]: https://github.com/olillevik/shield-up/issues/7
[os-sbx-default]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/sandboxes/overview.mdx#L241-L247
[os-sbx-env]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/sandboxes/overview.mdx#L382-L392
[os-sbx-restart]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/sandboxes/overview.mdx#L43-L48
[os-upgrade-image]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/upgrade/0-1-0.mdx#L55
[os-providers-readme]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/providers/README.md#L27-L40
[os-profile-claude]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/providers/claude-code.yaml#L12-L55
[os-profile-copilot]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/providers/copilot.yaml#L12-L73
[os-profile-github]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/providers/github.yaml#L12-L64
[os-prov-bare]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx#L55-L64
[os-prov-refresh-secret]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx#L224-L229
[os-prov-profileless]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx#L107-L110
[os-prov-locations]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx#L394-L409
[os-maintainers]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/MAINTAINERS.md#L15-L16
[os-prov-auto]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx#L344-L362
[os-prov-inject]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx#L364-L369
[os-prov-failclosed]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx#L411-L420
[os-prov-note]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/overview.mdx#L457-L459
[os-profiles-intro]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx#L11-L28
[os-profiles-check]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx#L49-L79
[os-profiles-catalog]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx#L252
[os-profiles-discovery]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx#L487-L491
[os-profiles-runtime]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/providers/profiles.mdx#L1119-L1131
[os-binary-matching]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/network-rules.mdx#L57-L77
[os-best-binary]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx#L80-L92
[os-best-tls]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx#L116-L127
[os-code-context]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-providers/src/context.rs#L4-L14
[os-code-discovery]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-providers/src/discovery.rs#L33-L67
[os-code-ensure]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-cli/src/commands/provider.rs#L409-L449
[os-code-placeholder]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/secrets.rs#L831-L845
[os-code-header]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/secrets.rs#L519-L555
[os-code-proxy-env]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-sandbox/src/process.rs#L125-L153
[os-code-tls-env]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-sandbox/src/child_env.rs#L6-L21
[os-code-unique-create]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/sandbox.rs#L536
[os-code-unique-attach]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/sandbox.rs#L1412
[os-e2e]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/e2e/rust/tests/provider_auto_create.rs#L73-L197
[os-620]: https://github.com/NVIDIA/OpenShell/issues/620
[os-1925]: https://github.com/NVIDIA/OpenShell/issues/1925
[os-3331]: https://github.com/NVIDIA/OpenShell/issues/3331
[community-dockerfile]: https://github.com/NVIDIA/OpenShell-Community/blob/edb65583a32c9ebb0fe8b6c709949e19ece08e52/sandboxes/base/Dockerfile
[community-readme]: https://github.com/NVIDIA/OpenShell-Community/blob/edb65583a32c9ebb0fe8b6c709949e19ece08e52/README.md
[claude-login]: https://code.claude.com/docs/en/authentication#log-in-to-claude-code
[claude-precedence]: https://code.claude.com/docs/en/authentication#authentication-precedence
[claude-storage]: https://code.claude.com/docs/en/authentication#credential-management
[claude-setup-token]: https://code.claude.com/docs/en/authentication#generate-a-long-lived-token
[claude-env]: https://code.claude.com/docs/en/env-vars
[claude-network]: https://code.claude.com/docs/en/network-config#network-access-requirements
[claude-ca]: https://code.claude.com/docs/en/network-config#custom-ca-certificates
[claude-webfetch]: https://code.claude.com/docs/en/data-usage#webfetch-domain-safety-check
[claude-update]: https://code.claude.com/docs/en/setup#update-claude-code
[claude-auto-update]: https://code.claude.com/docs/en/setup#auto-updates
[claude-disable-update]: https://code.claude.com/docs/en/setup#disable-auto-updates
[claude-pkg]: https://code.claude.com/docs/en/setup#install-with-linux-package-managers
[claude-devcontainer]: https://code.claude.com/docs/en/devcontainer#persist-authentication-and-settings-across-rebuilds
[copilot-auth]: https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/authenticate-copilot-cli#how-copilot-cli-stores-credentials
[copilot-tokens]: https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/authenticate-copilot-cli#supported-token-types
[copilot-auth-env]: https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/authenticate-copilot-cli#authenticating-with-environment-variables
[copilot-auth-oauth]: https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/authenticate-copilot-cli#authenticating-with-oauth
[copilot-auth-store]: https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/authenticate-copilot-cli#how-copilot-cli-stores-credentials
[copilot-unauth]: https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/authenticate-copilot-cli#unauthenticated-use
[copilot-classic]: https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/troubleshoot-copilot-cli-auth#token-classic-rejected
[copilot-install]: https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/install-copilot-cli
[copilot-env]: https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference#environment-variables
[copilot-options]: https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference#command-line-options
[copilot-mcp]: https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference#built-in-mcp-servers
[copilot-config]: https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-config-dir-reference#user-settings-copilotsettingsjson
[gh-allowlist]: https://docs.github.com/en/copilot/reference/copilot-allowlist-reference#copilot-on-githubcom
[gh-pat]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token
