# What an OpenShell 0.1.3 network policy can express

Research for #8, a ticket on the map #1. Investigated on 2026-10-09 against
OpenShell 0.1.3: the local `openshell` CLI help, and the NVIDIA/OpenShell
repository at tag `v0.1.3` (commit `e1f3c82`). Every source link points at that
tag. GitHub API facts come from GitHub's REST docs, its live GraphQL schema, and
read-only `gh api` calls made the same day. No sandbox, policy or setting was
created or changed.

## Short answer

A network policy is a map of named rules. Each rule pairs a list of endpoints
(a host or host wildcard, a port, and optional destination IP ranges) with a
list of executables. It allows every listed executable, and every process that
executable starts, to connect to every listed endpoint. OpenShell denies every
other connection. An endpoint can also turn on request inspection. With
`protocol: rest`, rules match the HTTP method, a path glob and query parameters.
With `protocol: graphql`, they match the operation type, operation name and
top-level field names that OpenShell parses out of the request body. MCP,
JSON-RPC and WebSocket endpoints have their own matchers. A matching deny rule
beats any allow.

No policy rule can match a REST request body. The only hook that sees a whole
body is supervisor middleware, a separate gRPC service registered in the
gateway's configuration that can deny any request the policy allowed. The one
built-in middleware redacts a fixed `sk-` pattern and is documented as a
demonstration.

Request inspection needs TLS interception, and OpenShell does it by default on
every endpoint that does not set `tls: skip`. Each sandbox gets an ephemeral CA,
and OpenShell points the usual trust-store environment variables at it.

For the PR approval question this means a policy can block the review
endpoints and mutations as a whole, but it cannot tell `event: APPROVE` apart
from `COMMENT` or `REQUEST_CHANGES`. Denying the two REST review endpoints and
the GraphQL mutations `addPullRequestReview` and `submitPullRequestReview`
stops approvals, and it also stops the agent from submitting any review at all.
Only a custom middleware could block approvals alone.

The rest of the question, briefly. Network rules and middleware can change on a
running sandbox; filesystem and process settings cannot. The built-in default
policy has no network rules. A policy belongs to one sandbox, unless a gateway
administrator sets a global policy, which replaces every sandbox's policy and
locks it. Agent proposals wait for a human by default, but a gateway-wide
`proposal_approval_mode` value overrides the sandbox's own.

## Rule structure and connection matching

`network_policies` is a map of named rules, each with `endpoints` and
`binaries`, and a rule allows every listed binary to reach every listed endpoint
([schema][schema-netpol], [network rules][rules-structure]). An endpoint names a
host, either exact or a wildcard with at least three DNS labels, plus `port` or
`ports`, and optionally `allowed_ips` ([schema][schema-dest]). Host matching is
case-insensitive. Binary and request path matching is case-sensitive
([schema][schema-matcher]). Loopback, link-local and unspecified destinations,
`169.254.169.254` among them, are always denied, and private addresses need an
exact hostname or `allowed_ips` ([schema][schema-dest]).

A binary matches when its path, as the kernel reports it, equals a listed path
or glob. It also matches when any ancestor process does
([network rules][rules-binary], [schema][schema-binary]). The Rego policy
engine checks globs against the executable and every ancestor
([sandbox-policy.rego][src-rego-binary]). A rule that lists the agent's
executable therefore covers every tool the agent starts. OpenShell pins each
executable's SHA-256 the first time it connects and denies later connections if
the file changes ([schema][schema-binary]). An empty `binaries` list matches
nothing ([network rules][rules-binary]).

The prover docs say "a sandbox runtime can turn enforcement off" for binary
identity ([prover][prover-binary]), and the Rego has a runtime switch under
which `binaries` is ignored entirely ([sandbox-policy.rego][src-rego-binary]).
I did not find which runtimes set it.

Rules are not an ordered list. Every matching rule adds access, and a matching
deny rule wins wherever it appears ([network rules][rules-overlap]). OpenShell
rejects a policy with an unknown field or a duplicate key
([schema][schema-top]), and the YAML structs are declared with
`deny_unknown_fields` ([policy-schema lib.rs][src-schema-structs]).

## Request rules by protocol

When an endpoint sets `protocol`, OpenShell reads each request on the
connection and checks it against the endpoint's `rules` (allow) and
`deny_rules` ([network rules][rules-checks]). `enforcement` defaults to
`audit`, which logs a violation and forwards the request anyway. `enforce`
blocks it ([schema][schema-inspect], [network rules][rules-enforcement]).
Without `protocol`, `access` and `rules` have no effect
([schema][schema-constraints]).

| `protocol` | What a rule can match | Source |
|---|---|---|
| `rest` | HTTP method or `*`, path glob, query parameter globs | [schema][schema-rest], [rego][src-rego-allow] |
| `graphql` | operation type, operation name glob, top-level field globs | [schema][schema-graphql] |
| `websocket` | upgrade `GET` or `WEBSOCKET_TEXT`, upgrade path, upgrade query | [schema][schema-ws] |
| `mcp` | MCP method, and tool name for `tools/call` | [schema][schema-mcp] |
| `json-rpc` | exact method name or `*` | [schema][schema-jsonrpc] |
| `tcp` | nothing beyond host, port and binary | [network rules][rules-checks] |

The access presets are shorthand. For REST, `read-only` is `GET`, `HEAD` and
`OPTIONS`, `read-write` adds `POST`, `PUT` and `PATCH`, and `full` allows every
method. For GraphQL they allow `query`, then `query` and `mutation`, then every
operation ([schema][schema-presets]). One endpoint takes `access` or `rules`,
and OpenShell rejects an endpoint that sets both ([schema][schema-constraints]).

In a REST path glob, `*` matches within one segment, and `**` written as a whole
segment matches one or more segments, so `/repos/**` does not match `/repos`
([schema][schema-matcher]). Before matching, the proxy canonicalises the path.
It resolves `.` and `..`, collapses repeated slashes, strips `;params` and
rejects `%2F` unless the endpoint sets `allow_encoded_slash`. The canonical path
is also what goes upstream ([path.rs][src-path]). A trailing slash survives
canonicalisation ([path.rs][src-path-trailing]).

Several endpoints can share a host and port with different `path` selectors,
and the most specific match wins. The docs use this to send `/graphql` on
`api.github.com` to GraphQL rules and every other path to REST rules
([network rules][rules-graphql]).

The source also accepts a `protocol: sql` with a `command` matcher that the docs
do not mention. Validation rejects it with `enforcement: enforce`, so it can
only audit ([policy lib.rs][src-sql]).

## Request bodies

Policy rules see no REST body. The rule structs have no body field
([policy-schema lib.rs][src-schema-structs]). The Rego REST matchers read only
`method`, `path` and `query_params` ([allow][src-rego-allow],
[deny][src-rego-deny]). The per-request record handed to the policy engine holds
the method, the path, the query parameters, and parsed GraphQL or JSON-RPC
metadata, and nothing else ([l7/mod.rs][src-l7-requestinfo]).

GraphQL, MCP and JSON-RPC inspection read the body, but only to pull out the
fields in the table above. For GraphQL the proxy parses the JSON envelope, which
can be a single request or a batch, picks the operation, and collects its
top-level field names ([graphql.rs][src-graphql]). It follows fragments, and it
records the schema field name, so an alias does not hide a field. It collects no
arguments or variables, which means `event: APPROVE` is invisible to a GraphQL
rule. One denied operation denies the whole batch ([schema][schema-graphql]).
MCP rules do not match tool arguments ([schema][schema-mcp]), and JSON-RPC rules
do not match parameters ([schema][schema-jsonrpc]).

Supervisor middleware is the only place a whole body reaches a decision.
OpenShell calls middleware after the policy allows a request and before it
injects provider credentials. The middleware receives the body, target and
headers, and it can allow the request, deny it, replace the body or change
headers ([middleware][mw-index], [operations][mw-ops-request]). A middleware
denial blocks the request even on an `audit` endpoint
([configuration][mw-configure-onerror]). A policy attaches middleware by host
pattern in `network_middlewares` ([schema][schema-middleware]). Anything other
than a built-in must be a gRPC service registered in the gateway's TOML, the
gateway must be restarted, and the gateway does not start while a registered
service is unreachable ([configuration][mw-configure-register]). The only
built-in, `openshell/regex`, replaces a fixed `sk-` pattern, and the docs say
not to rely on it ([configuration][mw-regex]). Middleware cannot see traffic to
`tls: skip` endpoints ([middleware][mw-index]).

## TLS interception

OpenShell terminates TLS itself. The proxy peeks at the first bytes of each
tunnel. On a TLS ClientHello it terminates TLS with a per-sandbox ephemeral CA,
reads the plaintext HTTP, and re-encrypts to the upstream against the real root
CAs ([best practices][bp-tls], [tls.rs][src-tls]). This happens on every
endpoint without `tls: skip`, whether or not it sets `protocol`. An endpoint
without `protocol` still has its TLS terminated and its HTTP parsed, and then
allows any method and path ([network rules][rules-checks],
[best practices][bp-l7]). `tls: skip` relays the encrypted bytes, which turns
off inspection, middleware and provider credential injection for that endpoint
([schema][schema-inspect], [best practices][bp-tls], [middleware][mw-index]).

For sandbox processes OpenShell sets `NODE_EXTRA_CA_CERTS`, `DENO_CERT`,
`SSL_CERT_FILE`, `REQUESTS_CA_BUNDLE`, `CURL_CA_BUNDLE` and `GIT_SSL_CAINFO`
([child_env.rs][src-child-env], [best practices][bp-tls]), and the workload can
read the CA under `/run/openshell-supervisor-ca`
([default policy][default-ca]). A client that ignores all of these, or pins
certificates, fails the handshake. The terminator advertises only `http/1.1` in
ALPN, to the client and to the upstream ([tls.rs][src-tls-alpn-client],
[tls.rs][src-tls-alpn-upstream]), and an `h2c` upgrade is refused with
`unsupported_l7_protocol` ([manage policies][manage-troubleshoot]).

## How allow and deny rules combine

A request is denied if a rule that matches its endpoint and binary has a
matching deny rule ([sandbox-policy.rego][src-rego-deny],
[network rules][rules-overlap]). The binary condition is easy to miss. The Rego
evaluates a rule's `deny_rules` only when that same rule's `binaries` match the
requesting process or one of its ancestors ([sandbox-policy.rego][src-rego-deny]).
A deny rule inside a rule that lists only `/usr/bin/gh` does nothing to `curl`
reaching the same path through another rule. For a deny to cover everything the
agent runs, the rule that carries it has to list the agent's executable, which
covers its descendants, or a glob that matches every executable.

`deny_rules` require `protocol`, and outside MCP they also require `rules` or
`access` on the same endpoint ([schema][schema-constraints]). A rule that
carries deny rules therefore always allows something as well.

An allow rule fails closed on a path spelling it does not foresee, and a deny
rule fails open. On 2026-10-09, `gh api /repos/OLILLEVIK/SHIELD-UP` returned
`olillevik/shield-up`, so GitHub matched the owner and repo case-insensitively
on that GET. OpenShell path matching is case-sensitive
([schema][schema-matcher]). A deny pattern that spells out the owner and repo
can therefore be dodged by changing their case, and `*` in both segments closes
that gap. `GET /repos/olillevik/shield-up/` and
`GET /repos/olillevik/shield-up/pulls/`, with trailing slashes, returned 404. I
did not try `POST`.

## Blocking PR approvals

GitHub has two REST endpoints whose body can carry `event: APPROVE`
([GitHub REST docs][gh-rest-reviews]). `POST
/repos/{owner}/{repo}/pulls/{pull_number}/reviews` creates a review, and
leaving `event` out creates a pending one. `POST
/repos/{owner}/{repo}/pulls/{pull_number}/reviews/{review_id}/events` submits a
pending review. In GitHub's GraphQL schema, introspected on 2026-10-09, only
`AddPullRequestReviewInput` and `SubmitPullRequestReviewInput` have a field of
type `PullRequestReviewEvent`, whose values are `COMMENT`, `APPROVE`,
`REQUEST_CHANGES` and `DISMISS`. They are the inputs of the mutations
`addPullRequestReview` and `submitPullRequestReview`.

An OpenShell policy can deny all four by method, path and GraphQL field. This
sketch follows the schema above and has not been run in a sandbox. A `/**`
binary glob matches every executable ([best practices][bp-binary]), so the deny
rules apply to any process that reaches `api.github.com`. It also lets any
process reach that host.

```yaml
network_policies:
  github_api:
    endpoints:
      - host: api.github.com
        port: 443
        protocol: rest
        enforcement: enforce
        access: full
        deny_rules:
          - method: POST
            path: /repos/*/*/pulls/*/reviews
          - method: POST
            path: /repos/*/*/pulls/*/reviews/*/events
      - host: api.github.com
        port: 443
        path: /graphql
        protocol: graphql
        enforcement: enforce
        access: full
        deny_rules:
          - operation_type: mutation
            fields: [addPullRequestReview, submitPullRequestReview]
    binaries:
      - path: /**
```

These rules block every review submission, `COMMENT` and `REQUEST_CHANGES`
included, and the map says the agent may comment and request changes. Paths
outside these patterns, such as issue comments, are not affected. To block
approvals alone, a middleware would have to parse REST JSON bodies for
`"event": "APPROVE"` and GraphQL bodies for `APPROVE` in inline arguments or in
`variables`. That is a gRPC service shield-up would have to run and register in
the user's gateway configuration ([configuration][mw-configure-register]).

The map lists "Path-level GitHub rules in the network policy" as out of scope.
Blocking approvals in the policy needs exactly such rules.

## Changing a running sandbox's policy

`openshell policy update` edits only `network_policies` on a live sandbox. It
adds, merges or removes endpoints, appends allow or deny rules written as
`host:port:METHOD:path_glob`, and removes rules by name (`openshell policy
update --help`, [manage policies][manage-update]). It cannot write middleware,
GraphQL, MCP or JSON-RPC rules, `tls: skip`, credential bindings or request
signing. Those need `openshell policy set`, which replaces the whole policy
([manage policies][manage-set]).

Network rules and middleware take effect while the sandbox runs, and OpenShell
closes connections opened under the old rules, so clients reconnect.
Filesystem, Landlock and process settings change only by recreating the sandbox
([manage policies][manage-effect], [overview][overview-sections]). The gateway
validates each change, and the sandbox validates it again. If the sandbox
cannot load it, the gateway's `policy_validation_failure_mode` decides what
happens, and the default `fail_closed` blocks network traffic until a valid
policy loads ([manage policies][manage-effect]). With `--wait`, the CLI waits
for the sandbox to report the load ([manage policies][manage-verify]). Every
change becomes a revision that `openshell policy list` shows, and a rollback
re-submits an old revision ([manage policies][manage-rollback]).

## Agent proposals and approval

The policy advisor is off by default, controlled by the
`agent_policy_proposals_enabled` setting ([advisor][advisor-intro],
[settings.rs][src-settings]). When it is on, the agent reads a guide at
`/etc/openshell/skills/policy_advisor.md` and submits proposals to
`http://policy.local` ([advisor][advisor-intro]). A proposal can only add a
network rule. It cannot remove rules or touch filesystem, Landlock or process
settings ([advisor][advisor-intro]). It can carry an access preset, method and
path allow rules, deny rules, `allowed_ips` and `allow_encoded_slash`. It
cannot carry `protocol: tcp`, `tls: skip`, GraphQL or MCP rules, query matchers,
credential settings or endpoint `path` selectors ([advisor][advisor-propose]).
OpenShell adds an approved proposal as a separate rule, and deny rules in other
rules still apply to it ([advisor][advisor-propose]), subject to the binary
condition described above.

Even with the advisor off, OpenShell drafts a proposal from every blocked
connection in every sandbox, each one allowing one binary to reach one host and
port ([advisor][advisor-drafts]).

In `manual` mode, the default, every proposal waits for a person to run
`openshell rule get <sandbox> --status pending` and then `openshell rule
approve` or `openshell rule reject` ([advisor][advisor-review]). The `rule`
command exists in 0.1.3 but is missing from the command list in
`openshell --help`. `openshell rule approve-all` approves every pending
proposal except security-flagged ones, which it approves only with
`--include-security-flagged` (`openshell rule approve-all --help`). In `auto`
mode, a proposal with no prover finding and no flagged destination is approved
without review, and the docs warn that this lets binaries reach new public
hosts unreviewed ([advisor][advisor-auto]).

The approval mode resolves gateway scope first, so a global
`proposal_approval_mode` overrides the sandbox's value ([advisor][advisor-auto],
[settings.rs][src-settings]). `openshell sandbox create --approval-mode manual`
writes no setting and relies on the default ([run.rs][src-cli-approval]), so it
does not protect a sandbox from a global `auto`. On this machine,
`openshell settings get --global` showed `agent_policy_proposals_enabled`,
`proposal_approval_mode` and both OCSF settings unset on 2026-10-09. While a
global policy is active, no proposal can be approved
([advisor][advisor-review], [overview][overview-global]). Approved rules last
across restarts of the same sandbox and are gone when it is recreated
([best practices][bp-approval]).

## The default policy

The built-in default applies only when there is no global policy, no saved
sandbox policy and no policy in the image at `/etc/openshell/policy.yaml`
([default policy][default-when], [manage policies][manage-create]). Its
`network_policies` map is empty, so all outbound traffic is denied
([policy lib.rs][src-default], [default policy][default-net]). It grants
read-only `/bin`, `/usr`, `/lib`, `/proc`, `/dev/urandom`, `/etc` and
`/var/log`, read-write `/tmp`, `/dev/null` and the working directory, and
Landlock `best_effort` ([policy lib.rs][src-default]). The supervisor saves it
to the gateway as the sandbox's base policy
([supervisor lib.rs][src-default-fallback]), and attached providers still add
their rules to the effective policy
([default policy][default-net], [overview][overview-effective]). If shield-up
passes `--policy`, the default does not apply to its sandboxes.

Provider rules are part of what a shield-up sandbox can reach. The example
GitHub provider profile in the repo contributes read-only REST and GraphQL on
`api.github.com`, and clone and fetch only on `github.com`, for `gh` and `git`
([github.yaml][provider-github]). Git's Smart HTTP traffic is inspected as
REST, so a policy can limit pushes to one repository's `git-receive-pack` path
([GitHub push tutorial][tutorial-push]).

## Scope of a policy

A policy belongs to one sandbox. `--policy` at creation, or the client-side
`OPENSHELL_SANDBOX_POLICY` variable, becomes the sandbox's saved policy, and
later changes revise it ([overview][overview-sources],
[manage policies][manage-create]). Sandboxes, policies and settings also belong
to a workspace, which is `default` unless `--workspace` is set
([workspaces][workspaces]).

A global policy is the exception. `openshell policy set --global` applies one
policy to every sandbox on the gateway. It replaces each sandbox's policy,
blocks sandbox policy changes and proposal approvals, and suppresses provider
rules until `openshell policy delete --global` removes it
([overview][overview-global], [manage policies][manage-global]). It needs the
Platform Admin role, and a local gateway without OIDC roles treats every
authenticated user as a Platform Admin ([manage policies][manage-global],
[workspaces][workspaces-local-admin]). On this machine on 2026-10-09,
`openshell policy get --global` reported that no global policy revision exists.

## Open questions

1. Which compute runtimes turn off binary identity enforcement, and does the
   local macOS gateway? With it off, `binaries` is ignored and every rule
   applies to every process ([prover][prover-binary]).
2. Do Claude Code, Copilot CLI and `gh` trust the sandbox CA through the
   environment variables OpenShell sets? A client that pins certificates or
   bundles its own roots would fail on every terminated endpoint. This belongs
   with #7.
3. Does GitHub accept `POST .../reviews/` with a trailing slash, or any other
   spelling the deny patterns above miss? Only `GET` was observed.
4. Can a process inside the sandbox reach the gateway API and change its own
   policy by some route other than `policy.local` proposals? None of the
   sources I read address it.
5. Should shield-up keep proposals approved in earlier sessions? They live in
   the sandbox's base policy ([advisor][advisor-propose]), and `policy set`
   replaces the base policy ([manage policies][manage-set]), so re-applying
   shield-up's policy on every reattach would drop them.
6. Does the proposal flow behave in a live sandbox as the docs and source
   describe? No sandbox was created to watch it.

[schema-top]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L11-L13
[schema-netpol]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L102-L115
[schema-dest]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L121-L149
[schema-inspect]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L151-L161
[schema-constraints]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L209-L226
[schema-presets]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L228-L237
[schema-rest]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L258-L281
[schema-ws]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L283-L304
[schema-graphql]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L306-L332
[schema-mcp]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L334-L353
[schema-jsonrpc]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L502-L517
[schema-binary]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L519-L531
[schema-middleware]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L533-L551
[schema-matcher]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/schema.mdx?plain=1#L566-L595
[rules-structure]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/network-rules.mdx?plain=1#L22-L55
[rules-binary]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/network-rules.mdx?plain=1#L57-L77
[rules-checks]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/network-rules.mdx?plain=1#L79-L113
[rules-enforcement]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/network-rules.mdx?plain=1#L115-L138
[rules-overlap]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/network-rules.mdx?plain=1#L140-L155
[rules-graphql]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/network-rules.mdx?plain=1#L450-L497
[manage-create]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/manage-policies.mdx?plain=1#L16-L52
[manage-update]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/manage-policies.mdx?plain=1#L84-L134
[manage-set]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/manage-policies.mdx?plain=1#L136-L141
[manage-effect]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/manage-policies.mdx?plain=1#L183-L221
[manage-verify]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/manage-policies.mdx?plain=1#L252-L275
[manage-rollback]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/manage-policies.mdx?plain=1#L277-L296
[manage-global]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/manage-policies.mdx?plain=1#L298-L319
[manage-troubleshoot]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/manage-policies.mdx?plain=1#L337-L343
[overview-sections]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/overview.mdx?plain=1#L27-L38
[overview-sources]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/overview.mdx?plain=1#L53-L67
[overview-effective]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/overview.mdx?plain=1#L69-L80
[overview-global]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/overview.mdx?plain=1#L82-L91
[default-when]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/default-policy.mdx?plain=1#L15-L24
[default-net]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/default-policy.mdx?plain=1#L42-L47
[default-ca]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/default-policy.mdx?plain=1#L84-L86
[advisor-intro]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/advisor.mdx?plain=1#L11-L74
[advisor-drafts]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/advisor.mdx?plain=1#L53-L59
[advisor-review]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/advisor.mdx?plain=1#L114-L151
[advisor-auto]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/advisor.mdx?plain=1#L153-L190
[advisor-propose]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/advisor.mdx?plain=1#L213-L277
[prover-binary]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/prover.mdx?plain=1#L209-L214
[bp-binary]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx?plain=1#L80-L92
[bp-l7]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx?plain=1#L94-L103
[bp-tls]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx?plain=1#L116-L127
[bp-approval]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/security/best-practices.mdx?plain=1#L140-L150
[mw-index]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/extensibility/supervisor-middleware/index.mdx?plain=1#L10-L58
[mw-ops-request]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/extensibility/supervisor-middleware/operations.mdx?plain=1#L43-L76
[mw-configure-register]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/extensibility/supervisor-middleware/configure.mdx?plain=1#L13-L63
[mw-regex]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/extensibility/supervisor-middleware/configure.mdx?plain=1#L123-L127
[mw-configure-onerror]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/extensibility/supervisor-middleware/configure.mdx?plain=1#L166-L178
[workspaces]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/workspaces.mdx?plain=1#L11-L18
[workspaces-local-admin]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/workspaces.mdx?plain=1#L60-L63
[tutorial-push]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/tutorials/github-push-access.mdx?plain=1#L141-L187
[provider-github]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/providers/github.yaml#L35-L64
[src-schema-structs]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-policy-schema/src/lib.rs#L382-L465
[src-rego-binary]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor-network/data/sandbox-policy.rego#L138-L172
[src-rego-deny]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor-network/data/sandbox-policy.rego#L282-L300
[src-rego-allow]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor-network/data/sandbox-policy.rego#L516-L526
[src-l7-requestinfo]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor-network/src/l7/mod.rs#L277-L290
[src-graphql]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor-network/src/l7/graphql.rs#L144-L338
[src-path]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor-network/src/l7/path.rs#L4-L32
[src-path-trailing]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor-network/src/l7/path.rs#L316-L320
[src-tls]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor-network/src/l7/tls.rs#L4-L9
[src-tls-alpn-client]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor-network/src/l7/tls.rs#L229
[src-tls-alpn-upstream]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor-network/src/l7/tls.rs#L285
[src-child-env]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-sandbox/src/child_env.rs#L6-L22
[src-default]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-policy/src/lib.rs#L965-L994
[src-default-fallback]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor/src/lib.rs#L2382-L2408
[src-sql]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-policy/src/lib.rs#L1628-L1632
[src-settings]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/settings.rs#L78-L107
[src-cli-approval]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-cli/src/run.rs#L746-L773
[gh-rest-reviews]: https://docs.github.com/en/rest/pulls/reviews
