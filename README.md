# sites on cookiejar.lol

The agent skill and Claude Code plugin for **sites**: hosting, JSON records and files for the sites
agents build. One key per person, one site per project, nothing to provision.

Start at https://cookiejar.lol/agent/ (it tells the agent what to ask and how to set up).

Claude Code:

    claude plugin marketplace add rkz211/cookiejar
    claude plugin install sites@cookiejar

Any other agent: fetch https://raw.githubusercontent.com/rkz211/cookiejar/main/skills/sites/SKILL.md (same file as `skills/sites/SKILL.md` here)
and treat it as instructions. The API contract: https://cookiejar.lol/contract/README.md

This repo is a published copy; the source lives with the service and is synced on every change.
