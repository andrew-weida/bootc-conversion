# Application Convertion for RHEL image mode (bootc)
An [Agent Skill](https://agentskills.io) that helps an AI coding agent analyze how an
existing application is installed and convert it to run in RHEL image mode (bootc).

Works with any tool that supports the Agent Skills standard — GitHub Copilot in VS Code,
Claude Code, Copilot CLI, and others.

## Contents

| Path | What it is |
|---|---|
| [SKILL.md](.github/skills/bootc-conversion/SKILL.md) | The conversion procedure the agent follows |
| [references/bootc-reference.md](.github/skills/bootc-conversion/references/bootc-reference.md) | Full bootc guide — filesystem semantics, Containerfile patterns, updates, security, anti-patterns |
| [references/validation.md](.github/skills/bootc-conversion/references/validation.md) | How to validate the resulting image, including whether this host can boot one |

The reference guide is compiled from the official bootc documentation, Red Hat's RHEL 9
and RHEL 10 image mode documentation, and the Red Hat Community of Practice demo
repository. It is worth reading on its own, with or without an agent.

## Install

Copy the skill folder into a skills directory. For a personal install available in every
project:

```bash
git clone https://github.com/<owner>/bootc-container-image-best-practices.git
mkdir -p ~/.claude/skills
cp -r bootc-container-image-best-practices/.github/skills/bootc-conversion ~/.claude/skills/
```

`~/.claude/skills/` is read by both Claude Code and VS Code. Equivalent personal
locations are `~/.copilot/skills/` and `~/.agents/skills/`.

To scope it to a single repository instead, copy the folder into that repo's
`.github/skills/`, `.claude/skills/`, or `.agents/skills/`.

## Use

Once installed, the agent loads the skill on its own when you ask for something it
covers:

> Look at how this application installs and tell me what it would take to run it in RHEL
> image mode.

Or invoke it directly with `/bootc-conversion` in VS Code chat.

## What it covers

- Deciding whether an application belongs in the OS image at all, or should be a bound
  container
- Auditing an RPM, install script, or Ansible role for things that break in a bootc build
- Classifying every path the application touches into `/usr`, `/etc`, `/var`, `/opt`, or
  `/run`
- Translating install-time actions — `systemctl enable --now`, `useradd`, `mkdir /var/...`,
  `firewall-cmd`, interactive prompts — into their image mode equivalents
- Writing the Containerfile and its sidecar files
- Migrating state from an existing deployment
- Validating the result, up to and including booting it
