# What can the sandbox's OPENSHELL_SANDBOX variable authorize on the gateway?

Research for [#25][issue-25], a child of the map [#1][issue-1]. The primary source is NVIDIA/OpenShell at tag `v0.1.3` (commit `e1f3c82`), read on 2026-10-10 from a shallow clone. The local `openshell` 0.1.3 binary was read with `strings`, and only to confirm method names. I made no call to a gateway and read no sandbox environment. Each claim links to the file and line range it comes from, and the links pin those lines to the tag.

## Answer

The variable authorizes nothing. The driver sets it to the sandbox's name, and no gateway code reads it. The gateway identifies a sandbox from a gateway-minted JWT in the `authorization` header, and takes the sandbox's identity from that token's `sandbox_id` claim. A sandbox name in a request is a lookup key, and the gateway checks it against the token before acting on it. Presenting the name therefore does not reach `UpdateConfig`, `ApproveDraftChunk`, `ApproveAllDraftChunks`, or any other method. The value is an identifier, not a credential.

The sandbox's own session matters for the same question too. The router accepts a sandbox token on 17 methods. Three of them, `UpdateConfig`, `GetSandboxConfig` and `GetDraftPolicy`, are `dual`. Through `UpdateConfig`, a sandbox token can persist a policy revision for its own sandbox. It cannot change global config or settings, and it cannot delete settings. It cannot approve or reject draft chunks, because those methods are bearer-only. The source does not settle whether a sandbox-written policy revision takes effect. That is the first open point below.

## How the driver sets the variable

`SANDBOX` is the constant `"OPENSHELL_SANDBOX"`. Its doc comment says it is the name of the sandbox, used for policy sync and identification ([sandbox_env.rs][src-env-const]). Docker writes `OPENSHELL_SANDBOX=<sandbox.name>` into the container environment ([docker env][src-docker-env]). Podman does the same ([podman env][src-podman-env]). The VM driver sets it on the supervisor command ([vm command][src-vm-cmd]) and in its environment map ([vm env][src-vm-env]). The Kubernetes driver writes it into the pod environment ([k8s env][src-k8s-env]). The value is the gateway record's name, a plain string. It has no signature, expiry, claims, scope or audience, and no principal is issued it.

Docker removes this name from the user-supplied environment map, so a template or spec cannot set it ([docker protected names][src-docker-protected]). The Docker driver README lists it under "The supervisor owns these security-critical variables" ([README][src-docker-readme]).

## What the supervisor and workload see

The supervisor reads the variable as its `--sandbox` argument ([supervisor argument][src-sup-arg]). It passes the name to the gateway when it fetches policy at startup ([supervisor fetch][src-sup-fetch]), when it fetches sandbox config ([config request][src-grpc-config]), and when `policy.local` submits a proposal ([policy.local submit][src-policy-local-submit], [client submit][src-grpc-submit]). The same name is the sandbox name in OCSF events ([OCSF context][src-sup-ocsf]).

The workload does not get the name. `apply_canonical_process_environment` removes every variable with the `OPENSHELL_` prefix from the inherited environment, then sets `OPENSHELL_SANDBOX` to `1` as a marker ([process.rs][src-proc-env]). The boundary exec path clears its environment and sets the same `1` ([boundary exec][src-boundary-env]). A unit test checks that the workload sees `1` and no other reserved variable ([test][src-proc-test]). The premise that sandbox processes can read the variable holds for the supervisor only. Workload processes read `1`. Whether a workload can read the supervisor's copy through `/proc` is an OS question, covered in the open points.

## How the gateway authenticates a sandbox

The gateway runs each request through an authenticator chain. The chain holds the peer, compute-driver, sandbox-JWT and OIDC authenticators, each one only when it is configured ([chain build][src-chain-build]). Each authenticator reads the `authorization: Bearer` value ([bearer credential][src-bearer]). An authenticator that does not recognize the value returns no principal, and the chain moves on ([chain][src-chain]).

The [sandbox-JWT authenticator][src-sj-auth] is the one that matters for a sandbox. It requires the header type `openshell-gateway-session+jwt` ([type check][src-sj-type]). It verifies an EdDSA signature and reads the claims `iss`, `sub`, `aud`, `iat`, `exp`, `jti`, `sandbox_id`, `runtime_generation`, `auth_epoch` and `component` ([claims][src-jwt-claims]). It then checks the token against the persisted sandbox record ([persisted check][src-persisted-call], [authorize_persisted][src-persisted]). The `sandbox_id` in that token is the sandbox's identity. The supervisor's gateway client adds the same bearer to every call ([interceptor][src-interceptor]). The token reaches the supervisor from `OPENSHELL_SANDBOX_TOKEN` or `OPENSHELL_SANDBOX_TOKEN_FILE` ([token loading][src-acquire]), which are different variables from `OPENSHELL_SANDBOX` ([variable names][src-env-token]).

The other authenticators also reject a name. The compute-driver authenticator applies only to `IssueSandboxToken` ([compute driver][src-cd-auth]). Docker, Podman and MXC return unimplemented for it ([docker][src-docker-auth], [podman][src-podman-auth], [mxc][src-mxc-auth]). Kubernetes sends its credential to a TokenReview ([k8s driver][src-k8s-auth]). The OIDC authenticator returns "invalid token" when the bearer has no decodable JWT header ([oidc][src-oidc-decode]), so a name sent as a bearer fails there.

Inside a request, a sandbox name is resolved to a row and then matched to the token. For a sandbox token, `resolve_sandbox_by_name_for_principal` finds the row by name and then calls `ensure_sandbox_scope`, which requires the row's ID to equal the token's `sandbox_id` ([resolve by name][src-resolve-by-name], [scope guard][src-guard]). A name that belongs to another sandbox is refused with "cross-sandbox access denied" ([scope guard][src-guard]).

## Method by method

The router decides first. It refuses a sandbox token on any method whose auth mode is not `sandbox` or `dual` ([router gate][src-router-gate], [method check][src-method-authz]). The modes come from `(openshell.options.v1.authorization)` annotations in `proto/openshell.proto`, which the gateway reads from the compiled descriptor ([descriptor lookup][src-descriptor]). Line numbers in the table below refer to that proto file.

| Method | Proto auth (line) | Sandbox token at router | What the handler does for a sandbox token |
|---|---|---|---|
| `UpdateConfig` | dual, `config:write`, admin (452) | Passes | Accepts only a policy payload for its own sandbox. Refuses global writes, setting writes, deletions and merge operations ([validate][src-validate-caller], [handler][src-update-config]) |
| `ApproveDraftChunk` | bearer, `config:write`, admin (661) | Refused | Not reached. A test asserts the refusal for this method ([test][src-sandbox-methods-test]) |
| `ApproveAllDraftChunks` | bearer, `config:write`, admin (681) | Refused | Not reached |
| `RejectDraftChunk` | bearer, `config:write`, admin (671) | Refused | Not reached |
| `ClearDraftChunks` | bearer, `config:write`, admin (709) | Refused | Not reached |
| `UpdateProvider` | bearer, `provider:write`, admin (359) | Refused | Not reached |
| `UpdateProviderProfiles` | bearer, `provider:write`, admin (339) | Refused | Not reached |
| `StopSandbox` | bearer, `sandbox:write`, user (181) | Refused | Not reached. A test asserts the refusal for this method ([test][src-sandbox-methods-test]) |
| `DeleteSandbox` | bearer, `sandbox:write`, user (172) | Refused | Not reached. A test asserts the refusal for this method ([test][src-sandbox-methods-test]) |

Policy set and update are not separate methods in 0.1.3. `UpdateConfigRequest` carries the policy payload in its `policy` field, alongside settings and merge operations ([message][src-p-updateconfig]). Global policy writes use the same RPC and need a Platform Admin. `require_platform_admin` refuses every sandbox principal ([global branch][src-update-global], [require_platform_admin][src-platform]). Through `UpdateConfig`, the only policy write a sandbox can make is the sandbox-scoped one described above. When the spec already holds a policy, the static fields of the new policy must match that baseline ([static check][src-static]).

## Other methods that change state

The router accepts 17 methods from a sandbox token. Fourteen are sandbox-only and three are dual. The table lists them with what each one changes. The router's comment says handlers still check the same sandbox on request bodies ([sandbox methods][src-sandbox-methods]). I did not check every handler individually.

| Method | Mode | Line | Effect |
|---|---|---|---|
| `ReportPolicyStatus` | sandbox | 482 | Records the load result of a policy version |
| `ReportEndpointStatus` | sandbox | 490 | Replaces the observed endpoint status of the sandbox |
| `ReportProviderReadiness` | sandbox | 499 | Records provider state for the current supervisor session |
| `ReportSandboxConfiguration` | sandbox | 507 | Registers the startup configuration |
| `GetSandboxProviderEnvironment` | sandbox | 513 | Reads the provider environment at supervisor startup |
| `ExchangeProviderSubjectToken` | sandbox | 522 | Issues an intermediate token for the calling supervisor |
| `PushSandboxLogs` | sandbox | 539 | Writes supervisor logs |
| `ConnectSupervisor` | sandbox | 552 | Opens the persistent supervisor session |
| `ReportMainProcessExit` | sandbox | 559 | Persists the main process result |
| `FinalizeMainProcessExit` | sandbox | 566 | Confirms that terminal delivery finished |
| `RelayStream` | sandbox | 584 | Relays bytes between the supervisor and the gateway |
| `SubmitPolicyAnalysis` | sandbox | 644 | Writes draft chunks. The gateway auto-approves them when the resolved approval mode allows it ([auto-approve][src-submit-auto], [mode resolver][src-resolve-mode]) |
| `IssueSandboxToken` | sandbox | 732 | Mints a gateway JWT from a bootstrap credential |
| `RefreshSandboxToken` | sandbox | 744 | Renews the calling sandbox's JWT |
| `UpdateConfig` | dual | 452 | Persists a policy revision for its own sandbox (above) |
| `GetSandboxConfig` | dual | 427 | Reads config. Backfills policy revision 1 from the spec when none exists ([backfill][src-get-backfill]) |
| `GetDraftPolicy` | dual | 652 | Reads draft chunks ([handler][src-get-draft]) |

The other 60 methods are bearer-only, so the router refuses a sandbox token for each of them. They include sandbox lifecycle and access (`CreateSandbox`, `StartSandbox`, `StopSandbox`, `DeleteSandbox`, `ExecSandbox`, `ExecSandboxInteractive`, `ForwardTcp`, `CreateSshSession`, `ExposeService`, `AttachSandboxProvider`), provider writes (`CreateProvider`, `UpdateProvider`, `ImportProviderProfiles`, `UpdateProviderProfiles`, `DeleteProvider`, `RotateProviderCredential`), draft chunk writes (`ApproveDraftChunk`, `RejectDraftChunk`, `ApproveAllDraftChunks`, `EditDraftChunk`, `UndoDraftChunk`, `ClearDraftChunks`) and workspace membership (`CreateWorkspace`, `DeleteWorkspace`, `AddWorkspaceMember`, `RemoveWorkspaceMember`). The annotations are in [`proto/openshell.proto`][src-proto-file]. The four peer-only methods are refused to a sandbox token as well ([method check][src-method-authz]). `Health` is unauthenticated.

## Credential or identifier

The value is an identifier. It is the sandbox name copied from the gateway record ([docker env][src-docker-env]), with no signature, expiry or scope. Three checks support that reading. The gateway does not read it. A search of the repository for the exact string `OPENSHELL_SANDBOX` finds no hit in the gateway crates. Every use is in the supervisor, the sandbox runtime, a compute driver, the VM init script or the documentation ([sandbox_env users][src-env-const]). No client sends it, because the only bearer the client attaches is the session token ([interceptor][src-interceptor]). The host side does not read it either. The local CLI binary contains the string `OPENSHELL_SANDBOX_POLICY` and no `OPENSHELL_SANDBOX` string.

The credential the gateway checks is the session JWT described above. The variable does not travel with that token.

## Verdict and open points

Verdict: answered. The source settles the three questions the ticket asks. The variable authorizes no method, the gateway does not read it, and it is an identifier. Four points stay open. The first three concern what a sandbox can do with the token it holds, and the fourth concerns how far the workload check reaches. None of them changes the answer about the variable.

First, whether a workload process can read the supervisor's environment through `/proc/<supervisor pid>/environ`. The source sets the workload's environment to `1`, but it does not settle the OS permissions on `/proc`. Those depend on the runtime's UID and capability setup, which I did not trace. The live check is a permission test inside a running sandbox: confirm whether the workload user can open that file, without printing its contents.

Second, whether a workload process can read the sandbox JWT file. The Docker driver writes the token to a host state directory with owner-only permissions ([token writer][src-docker-token]). I did not trace the bind mount or UID mapping into the container, and I did not trace the Podman, VM or Kubernetes paths. The JWT is the credential that the 17 sandbox methods check. The live check is a readability test on the file named by `OPENSHELL_SANDBOX_TOKEN_FILE`, run from a workload process without printing it. The current policy blocks the sandbox from reaching the gateway, so this point matters only if that policy changes.

Third, whether a sandbox-written policy revision takes effect. A sandbox token can persist a new policy revision for its own sandbox ([sandbox branch][src-sandbox-policy], [persist][src-persist-rev]). The credential-policy check runs only for user callers ([caller skip][src-caller-skip]). The code comment says activation depends on `GetSandboxConfig` and `ReportSandboxConfiguration` ([comment][src-caller-skip]), and I did not trace those two gates. This is the path that decides whether a sandbox can change its own policy, so it needs its own ticket. Settling it in a live check would mean writing a policy as the supervisor on a test sandbox and reading the effective policy back, which this ticket does not do.

Fourth, the scope of the workload check. I traced the spawn paths in `process.rs` and `boundary_exec.rs`. I did not audit every helper binary the supervisor starts, or check each one for environment clearing.

## Sources

Primary: NVIDIA/OpenShell at tag `v0.1.3`, cloned with `gh repo clone` and `--depth 1`. The tag points at commit `e1f3c82caa3ed3b65de22889ae7ef32a774878ef`. File paths in this document are relative to that repository root. Local binary: `/opt/homebrew/Cellar/openshell/0.1.3/bin/openshell`, read with `strings`. Of the 82 RPC names in `proto/openshell.proto`, 62 appear as strings in the binary. The 20 absent names are the 14 sandbox-only methods, the four peer methods, `EditDraftChunk` and `UndoDraftChunk`. I did not check why those 20 are absent. No other source was used.

[issue-25]: https://github.com/olillevik/shield-up/issues/25
[issue-1]: https://github.com/olillevik/shield-up/issues/1
[src-proto-file]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/proto/openshell.proto
[src-p-updateconfig]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/proto/openshell.proto#L2696-L2737
[src-env-const]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/sandbox_env.rs#L14-L15
[src-env-token]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/sandbox_env.rs#L200-L210
[src-docker-env]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-driver-docker/src/lib.rs#L5078
[src-docker-protected]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-driver-docker/src/lib.rs#L5515-L5542
[src-docker-readme]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-driver-docker/README.md#L188-L193
[src-docker-token]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-driver-docker/src/lib.rs#L4188-L4222
[src-docker-auth]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-driver-docker/src/lib.rs#L3199-L3207
[src-podman-env]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-driver-podman/src/container.rs#L660-L662
[src-podman-auth]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-driver-podman/src/grpc.rs#L53-L65
[src-mxc-auth]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-driver-mxc/src/grpc.rs#L52-L59
[src-vm-cmd]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-driver-vm/src/driver.rs#L955
[src-vm-env]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-driver-vm/src/driver.rs#L5941-L5944
[src-k8s-env]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-driver-kubernetes/src/sandbox_runtime.rs#L246
[src-k8s-auth]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-driver-kubernetes/src/driver.rs#L863-L880
[src-sup-arg]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor/src/main.rs#L67-L68
[src-sup-fetch]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor/src/lib.rs#L2331-L2343
[src-sup-ocsf]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor/src/lib.rs#L742
[src-grpc-config]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/grpc_client.rs#L955-L958
[src-grpc-submit]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/grpc_client.rs#L1575-L1598
[src-policy-local-submit]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-supervisor-network/src/policy_local.rs#L553-L555
[src-proc-env]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-sandbox/src/process.rs#L204-L226
[src-proc-test]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-sandbox/src/process.rs#L1141-L1204
[src-boundary-env]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-sandbox/src/boundary_exec.rs#L158-L163
[src-chain-build]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/multiplex.rs#L901-L924
[src-router-gate]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/multiplex.rs#L1058-L1078
[src-bearer]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/authenticator.rs#L54-L59
[src-chain]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/authenticator.rs#L83-L94
[src-sj-type]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/sandbox_jwt.rs#L286-L288
[src-sj-auth]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/sandbox_jwt.rs#L274-L304
[src-persisted-call]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/sandbox_jwt.rs#L292-L295
[src-persisted]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/sandbox_session.rs#L289-L300
[src-jwt-claims]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/jwt.rs#L166-L177
[src-interceptor]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/grpc_client.rs#L187-L203
[src-acquire]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-core/src/grpc_client.rs#L353-L392
[src-cd-auth]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/compute_driver.rs#L13-L63
[src-oidc-decode]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/oidc.rs#L803-L806
[src-resolve-by-name]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L2662-L2699
[src-guard]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/guard.rs#L30-L54
[src-method-authz]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/method_authz.rs#L70-L78
[src-descriptor]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/descriptor_authz.rs#L237-L240
[src-sandbox-methods]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/sandbox_methods.rs#L4-L9
[src-sandbox-methods-test]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/sandbox_methods.rs#L38-L54
[src-validate-caller]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L2545-L2567
[src-update-config]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L3738-L3779
[src-update-global]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L3747-L3753
[src-platform]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/auth/workspace_authz.rs#L220-L234
[src-static]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/validation.rs#L1035-L1058
[src-sandbox-policy]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L4194-L4217
[src-caller-skip]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L4258-L4268
[src-persist-rev]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L4311-L4330
[src-resolve-mode]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L1499-L1530
[src-submit-auto]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L5428-L5440
[src-get-backfill]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L2836-L2870
[src-get-draft]: https://github.com/NVIDIA/OpenShell/blob/v0.1.3/crates/openshell-server/src/grpc/policy.rs#L5480-L5541
