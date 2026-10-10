# Can a gateway-wide OpenShell setting override manual approval of policy proposals?

Research for [#22][issue-22], a child of the map [#1][issue-1]. The sources are the local `openshell` 0.1.3 CLI and its gateway at `https://localhost:17670`, used read-only, and the `NVIDIA/OpenShell` repo at tag `v0.1.3` (commit `e1f3c82`), read on 2026-10-10. OpenShell links are pinned to the tag, and the docs I cite are the docs source at that tag, not the live site. The CLI calls a gateway-wide setting a global setting (`--global`), and this file does too. Each claim links to the page or file it comes from, and the text says so where the sources are silent or disagree. The local gateway had no sandboxes and I changed nothing on it, so every statement about a write or a running sandbox comes from source and docs, not from a run.

## Answer

Yes, one global setting can. `proposal_approval_mode` takes `manual` or `auto`, and the gateway reads the global value first, the sandbox value second and the default `manual` last ([resolver][src-resolve], [setting docs][src-settings-keys]). A global `auto` therefore beats a sandbox that asked for `manual`. `openshell sandbox create --approval-mode manual` is the CLI default and writes no setting, so it gives no protection against a global `auto` ([run.rs][src-cli-approval]).

The same setting keeps `manual` in force. A global `proposal_approval_mode` of `manual` pins every sandbox on the gateway: the gateway ignores sandbox values while a global value exists and refuses new sandbox-scope writes ([resolver][src-resolve], [update path][src-update-managed]). The OpenShell docs describe this use ([Policy Advisor][os-advisor-global-auto]). A Platform Admin can still change or delete a global value, and a gateway with no OIDC configured, such as the local one, treats every authenticated user as Platform Admin ([Workspaces][os-workspaces-ops]). A process inside the sandbox cannot change settings ([sandbox caller check][src-sandbox-caller]).

No other global setting changes the approval mode. `agent_policy_proposals_enabled` decides whether the agent gets the `policy.local` API. The two OCSF settings change logging. A global policy blocks every approval, human and automatic. A gateway interceptor, registered in the gateway TOML, can deny a write of `auto`, and OpenShell ships an example that does. I found no gateway TOML or Helm key that sets an approval mode. The table under [Other global settings](#other-global-settings) has the evidence for each.

For shield-up this means two checks and one optional write. Before it creates a sandbox, it reads `openshell settings get --global --json`. If the value is `auto` it stops. If the value is unset it can write `manual` globally to pin it. After it creates the sandbox, it reads `openshell settings get <sandbox> --json` and requires `manual` or unset. Neither step stops a Platform Admin from changing the setting later, and the next proposal follows the new value.

## How the mode resolves

The settings registry has four keys: `ocsf_json_enabled`, `ocsf_schema_version`, `agent_policy_proposals_enabled` and `proposal_approval_mode` ([settings.rs][src-settings-registry]). `openshell settings get --global` on the local gateway printed exactly those four, all `<unset>`, at settings revision 0. `openshell policy get --global` reported `no global policy revision found`.

`resolve_proposal_approval_mode` loads the global settings first. If the global map has the key, it returns whether the value is exactly `auto` and stops there. Only when the global map lacks the key does it read the sandbox's settings, and when both lack it the result is manual ([resolver][src-resolve]). Any string other than `auto` counts as manual, although `settings set` accepts only `manual` and `auto` ([settings.rs][src-settings-keys], [unknown-value test][src-test-unknown]). A unit test seeds a global `manual` and a sandbox `auto` and expects the proposal to stay pending ([test][src-test-global-manual]). The docs say the same: "A gateway-wide value overrides sandbox values, so an administrator can require manual review for every sandbox by setting `manual` on the gateway" ([Policy Advisor][os-advisor-global-auto]).

The gateway decides when a proposal arrives. The only call to `auto_approve_chunk` sits in the submit handler, after the proposal is stored ([submit gate][src-submit-gate], [auto_approve_chunk][src-auto-chunk]), so a change applies to proposals submitted after it. The sandbox does not need a restart or a config poll for that.

Once a global value exists, a sandbox-scope write or delete of the key fails with `setting 'proposal_approval_mode' is managed globally` ([update path][src-update-managed]). `sandbox create --approval-mode auto` creates the sandbox first and writes the setting afterwards, and a failed write prints a warning and leaves the sandbox running on the global value ([run.rs][src-cli-approval]). A sandbox-scope `auto` written before the global value was set stays in the store. It is ignored while the global value exists and applies again if the global value is deleted. That last point is my reading of the code, where the global branch returns before the sandbox settings are read.

Two sources disagree with the resolver or with the CLI. The `--approval-mode` help text and the `openshell-cli` skill describe `manual` as if it guarantees review, and neither mentions the global override ([CLI help][src-cli-help], [openshell-cli skill][skill-cli]). The resolver decides. Separately, the agent-facing guide and the CLI's own retry hint after a failed write use `openshell settings set <name> proposal_approval_mode auto`, but `openshell settings set --help` in 0.1.3 requires `--key` and `--value` ([guide][skill-advisor], [run.rs][src-cli-approval]). A script has to use the flags.

## What auto mode approves

In `auto` the gateway approves a proposal when the prover finds no new risk and OpenShell flags no destination. The docs add: "These checks do not treat access to a new public host as a risk when no provider credential applies there" ([Policy Advisor][os-advisor-auto]). The prover counts the credentials of attached providers, and a rule that extends a credential's reach or adds a method where a credential is already used produces a finding that waits for review ([Policy Advisor][os-advisor-prover]).

OpenShell also drafts proposals from connections it blocks, in every sandbox, whether or not the policy advisor is on, and automatic approval applies to those drafts too ([Policy Advisor][os-advisor-drafts], [mechanistic test][src-test-mechanistic]). Turning the policy advisor off therefore does not remove the need to pin `manual`.

## Who can change it

A global write goes through `require_platform_admin` ([update path][src-update-global]), and the operations table lists global configuration as Platform Admin only ([Workspaces][os-workspaces-ops]). A sandbox-scope write needs Workspace Admin on that sandbox's workspace ([proto][src-proto-updateconfig]). Platform Admin means the OIDC role named in `admin_role`, and the check passes for everyone when no OIDC table is configured ([admin_role][src-admin-role], [is_platform_admin][src-platform-admin]). The docs say it directly: "Local gateways without OIDC role configuration treat authenticated users as Platform Admins" ([Workspaces][os-workspaces-ops]).

The local gateway starts from `/opt/homebrew/var/openshell/gateway.toml`, which holds only `[openshell] version = 2` and an empty `[openshell.gateway]` table, and from `~/.config/openshell/gateway.env`, which sets only the Podman driver and its socket. By the rule above the local CLI user is a Platform Admin, even though `openshell whoami` lists only the `openshell-user` role. I did not run a write to confirm.

A sandbox principal cannot change the setting. `validate_sandbox_caller_update` rejects global writes, setting writes and deletes from sandbox callers and allows only a sandbox policy sync ([sandbox caller check][src-sandbox-caller]). `ApproveDraftChunk` is not callable by a sandbox ([test][src-sandbox-callable-test]). Reading global settings needs only an authenticated user, plus the `config:read` scope where scope enforcement is on ([proto][src-proto-getconfig], [test][src-test-getconfig]). shield-up can therefore check the value without Platform Admin.

## Other global settings

| Setting | Can it change who approves? | Evidence |
|---|---|---|
| `proposal_approval_mode` | Yes. A global value wins over any sandbox value, and `manual` pins | [resolver][src-resolve], [Policy Advisor][os-advisor-global-auto] |
| `agent_policy_proposals_enabled` | No. It turns the `policy.local` API and agent guide on or off, and OpenShell still drafts proposals when it is off | [settings.rs][src-settings-keys], [Policy Advisor][os-advisor-disabled], [supervisor][src-supervisor-proposals] |
| `ocsf_json_enabled`, `ocsf_schema_version` | No. They set the supervisor's OCSF log output | [settings.rs][src-settings-registry] |
| Global policy (`policy set --global`) | Not the mode, but while one is active nothing can be approved, human or automatic | [Policy Advisor][os-advisor-global-policy], [require_no_global_policy][src-no-global-policy], [auto_approve_chunk][src-auto-chunk] |
| Gateway TOML `[[openshell.gateway.interceptors]]` | Can deny writes. `UpdateConfig`, `SubmitPolicyAnalysis` and the approve and reject RPCs are on the interceptable list | [Gateway Interceptors][os-interceptors], [routes.rs][src-routes] |
| Gateway TOML `[openshell.gateway.oidc]`, `mtls_auth`, `auth` | Not the mode. They decide who may write it | [Configuration][os-config-auth], [Workspaces][os-workspaces-ops] |
| Other gateway TOML and Helm keys | None found | `proposal_approval_mode` appears only in the registry, resolver, CLI create path, docs, example interceptor and tests; `deploy/` has no hit for "proposal" |
| Global provider profiles (`profile import --global`) | Not the mode. Under `auto` they change what the prover flags, because it counts attached providers' credentials | [Policy Advisor][os-advisor-prover] |

The local gateway lists no interceptor. `openshell gateway info` shows three extensions, the `podman` compute driver, `openshell-driver-db-credstore` and `openshell/regex` supervisor middleware.

The example interceptor shows what a preventive control looks like. Among other RPCs it binds `UpdateConfig` and `SubmitPolicyAnalysis`, denies any write of `proposal_approval_mode` with the value `auto` at both scopes, and denies sandbox-authored proposals outright ([README][ex-readme], [example source][ex-main], [smoke test][ex-smoke]). The README says the proposal rule belongs to the example. Registration is static, so adding an interceptor needs a gateway restart and a running gRPC service ([Gateway Interceptors][os-interceptors-static]).

## What shield-up can check and set

These are the commands. I ran only the global read. The sandbox read needs a sandbox, and I did not run the write.

```shell
openshell settings get --global --json
openshell settings set --global --key proposal_approval_mode --value manual --yes
openshell settings get <sandbox> --json
```

The global read prints `"<unset>"` for a setting with no value, as the local gateway showed. The sandbox read prints each setting with a `value` and a `scope` of `global`, `sandbox` or `unset` ([run.rs][src-cli-settings]). `--yes` is required without a terminal. With a terminal the CLI asks first: "Setting 'proposal_approval_mode' globally will disable sandbox-level management for this key. Continue?" ([common.rs][src-cli-confirm]).

| Global value | `proposal_approval_mode` in the sandbox read | Meaning |
|---|---|---|
| `manual` | `manual`, scope `global` | Pinned. A sandbox-scope write fails |
| unset | `<unset>`, scope `unset` | Manual by default. A Workspace Admin can still write `auto` at sandbox scope, and a Platform Admin can write it globally |
| `auto` | `auto`, scope `global` | Automatic approval is on for every sandbox |
| unset | `auto`, scope `sandbox` | Automatic approval is on for this sandbox |

A global write changes the gateway for every sandbox, current and future. It also turns off `--approval-mode auto` and per-sandbox writes of the key for all of them. Whether shield-up writes a gateway-wide setting on a gateway the user shares with other work, or only checks it and refuses, is a design choice that this research does not settle.

A check cannot see a later change. The gateway logs each automatic approval as `CONFIG:APPROVED` with `auto:true`, the proposal `source` and `resolved_from` ([Policy Advisor][os-advisor-logs]), which lets a person find one afterwards. Short of limiting who holds Platform Admin, only an interceptor on `UpdateConfig` can stop the write.

## Effects on other tickets

The note on [#8][issue-8] says a global `proposal_approval_mode` overrides the sandbox value and that `--approval-mode manual` writes nothing. The resolver and its tests confirm both. This file adds that a global `manual` is the supported pin and that the setting is the only global one that matters.

The closed ticket [#12][issue-12] moved its two probes to [#14][issue-14]. The source here limits a sandbox principal to policy sync and bars it from approving, so a process in the sandbox has no route to the approval mode through the settings API. Whether the process can reach the gateway at all, and whether a policy sync can widen network rules, are probes for #14 that I did not examine.

The design note that proposals stay on `manual` needs the two checks above, and a decision on whether shield-up writes the global value.

## Open questions

1. Does `settings set --global` succeed for the local CLI user? The docs and code say yes, because no OIDC is configured, but only a write can show it.
2. After a global `manual` write, does `settings get <sandbox> --json` show `manual` with scope `global`? The CLI source says so, and the local gateway has no sandbox to read.
3. Does `sandbox create --approval-mode auto` against a global `manual` finish with only the warning? The CLI source says so.
4. Can a sandbox policy sync change network rules? The handler accepts a full policy from a sandbox caller, strips provider-derived entries and compares the static fields with the baseline ([update path][src-sync-policy]). I did not trace whether network rules may change. It belongs to #14.
5. Does the setting survive upgrades? NVIDIA/OpenShell issue [#2109][os-2109] and draft PR [#2168][os-2168] propose managed maximum permission modes, and the issue text says the older proposal-specific auto-approval overlaps with them. Neither is in 0.1.3. Re-read the setting's behavior when shield-up raises its minimum OpenShell version.

[issue-1]: https://github.com/olillevik/shield-up/issues/1
[issue-8]: https://github.com/olillevik/shield-up/issues/8
[issue-12]: https://github.com/olillevik/shield-up/issues/12
[issue-14]: https://github.com/olillevik/shield-up/issues/14
[issue-22]: https://github.com/olillevik/shield-up/issues/22
[os-advisor-drafts]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/advisor.mdx?plain=1#L53-L59
[os-advisor-global-policy]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/advisor.mdx?plain=1#L146-L151
[os-advisor-auto]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/advisor.mdx?plain=1#L153-L170
[os-advisor-global-auto]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/advisor.mdx?plain=1#L172-L190
[os-advisor-prover]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/advisor.mdx?plain=1#L192-L211
[os-advisor-disabled]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/advisor.mdx?plain=1#L295-L297
[os-advisor-logs]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/policies/advisor.mdx?plain=1#L299-L314
[os-workspaces-ops]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/workspaces.mdx?plain=1#L22-L64
[os-interceptors]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/extensibility/gateway-interceptors.mdx?plain=1#L14-L38
[os-interceptors-static]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/extensibility/gateway-interceptors.mdx?plain=1#L102
[os-config-auth]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/docs/how-it-works/gateways/configuration.mdx?plain=1#L302-L318
[src-settings-keys]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/settings.rs#L78-L107
[src-settings-registry]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/settings.rs#L109-L150
[src-resolve]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L1499-L1530
[src-submit-gate]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L5113-L5120
[src-auto-chunk]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L1753-L1770
[src-update-global]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L3747-L3756
[src-update-managed]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L4005-L4056
[src-sync-policy]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L4194-L4232
[src-sandbox-caller]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L2543-L2566
[src-no-global-policy]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L6778-L6785
[src-test-global-manual]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L17995-L18040
[src-test-unknown]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L17718-L17725
[src-test-mechanistic]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L16509-L16513
[src-test-getconfig]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L23759-L23780
[src-cli-approval]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-cli/src/run.rs#L746-L773
[src-cli-help]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-cli/src/main.rs#L1593-L1603
[src-cli-settings]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-cli/src/run.rs#L5227-L5362
[src-cli-confirm]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-cli/src/commands/common.rs#L671-L695
[src-admin-role]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/lib.rs#L432-L435
[src-platform-admin]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/workspace_authz.rs#L220-L246
[src-proto-updateconfig]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/proto/openshell.proto#L451-L459
[src-proto-getconfig]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/proto/openshell.proto#L436-L449
[src-sandbox-callable-test]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/sandbox_methods.rs#L38-L55
[src-routes]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-gateway-interceptors/src/routes.rs#L15-L43
[src-supervisor-proposals]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor/src/lib.rs#L4965-L4990
[ex-readme]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/examples/governance-interceptor/README.md#L24
[ex-main]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/examples/governance-interceptor/src/main.rs#L684-L756
[ex-smoke]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/examples/governance-interceptor/smoke.sh#L476
[skill-advisor]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor-process/src/skills/policy_advisor.md#L130-L140
[skill-cli]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/skills/openshell-cli/SKILL.md#L665
[os-2109]: https://github.com/NVIDIA/OpenShell/issues/2109
[os-2168]: https://github.com/NVIDIA/OpenShell/pull/2168
