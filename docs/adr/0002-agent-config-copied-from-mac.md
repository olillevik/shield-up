# Agent config is copied from this Mac at each launch

Shield-up never mounts or uploads a folder from the user's machine into a sandbox, with one exception: agent config, meaning the skills folder, `AGENTS.md` (as the sandbox's `CLAUDE.md`), and `minimalist-engineering.md`. Those are copied from the Mac at each launch, overwriting what is there, because a clone from GitHub would miss untracked and uncommitted skill files. The rule that uncommitted changes never leave the Mac covers the target repo's files only.
