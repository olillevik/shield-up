# shield-up

shield-up runs an agent inside an OpenShell sandbox against one GitHub repository, with a credential and a network policy scoped to that repository.

## Language

**Agent**:
The AI coding assistant working inside the sandbox: GitHub Copilot CLI or Claude Code.
_Avoid_: assistant, bot

**Sandbox**:
The OpenShell sandbox one agent runs in.
_Avoid_: container, VM, shell

**Sandbox image**:
The container image a target repo's sandbox is created from. Each target repo has its own, and the user edits it.
_Avoid_: profile, template

**Network policy**:
The OpenShell allowlist of hosts a sandbox may reach.
_Avoid_: DNS rules, firewall rules

**Target repo**:
The one GitHub repository a sandbox is created for and the repo credential is scoped to.
_Avoid_: project, workspace

**Agent credential**:
The credential the agent uses to reach its own model service: an Anthropic credential for Claude Code, or a GitHub Copilot credential for Copilot CLI. It is separate from the repo credential.
_Avoid_: agent token

**Agent config**:
The instructions and skills an agent reads, taken from the user's machine. Not the settings file, MCP servers, or credentials.

**Repo credential**:
The GitHub credential an agent uses for one repository. It can push, open pull requests and review, has no admin permission, and cannot bypass the repository's rules.
_Avoid_: PAT, token, GitHub token (those name a mechanism, not the concept)
