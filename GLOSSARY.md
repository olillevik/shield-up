# shield-up

shield-up runs an agent inside an OpenShell sandbox against one GitHub repository, with a credential and a network policy scoped to that repository.

## Language

**Agent**:
The AI coding assistant working inside the sandbox: GitHub Copilot CLI or Claude Code.
_Avoid_: assistant, bot

**Sandbox**:
The OpenShell sandbox one agent runs in.
_Avoid_: container, VM, shell

**Network policy**:
The OpenShell allowlist of hosts a sandbox may reach.
_Avoid_: DNS rules, firewall rules

**Target repo**:
The one GitHub repository a sandbox is created for and the repo credential is scoped to.
_Avoid_: project, workspace

**Agent credential**:
The login the agent uses to reach its own model service, Anthropic or GitHub Copilot. It is separate from the repo credential.
_Avoid_: agent token, API key

**Repo credential**:
The GitHub credential an agent uses for one repository. It can push, open pull requests and review, has no admin permission, and cannot bypass the repository's rules.
_Avoid_: PAT, token, GitHub token (those name a mechanism, not the concept)
