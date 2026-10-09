# Repo credential carries Workflows write

The repo credential carries Workflows write, so the agent can push changes under `.github/workflows/`, and the alternative keeps every CI change with a person. The grant also lets a workflow the agent pushes read the target repo's workflow secrets from a GitHub runner, a route that a sandbox does not control and shield-up does not address. The target repo's GitHub settings must close the bypass paths the grant opens, since shield-up checks neither.
