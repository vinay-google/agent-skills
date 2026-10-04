# Agent Skills (`SKILL.md`)

A collection of reusable public **Agent Skills** (`SKILL.md`) for AI coding agents such as **Antigravity CLI (`agy`)** and **Gemini CLI**.

## Available Skills

| Skill | Path | Description |
| :--- | :--- | :--- |
| **[`workspace-studio-node-addon`](skills/workspace-studio-node-addon/SKILL.md)** | [`skills/workspace-studio-node-addon/SKILL.md`](skills/workspace-studio-node-addon/SKILL.md) | End-to-end automation skill for building, deploying, and upgrading HTTP Google Workspace Studio Add-ons (Custom Starter Steps and Custom Action Steps) in Node.js on Cloud Run with Cloud Firestore and the Gemini Enterprise Agent Platform. |

## Quick Install

To install a skill into your project's `.agents/skills/` directory:

```bash
mkdir -p .agents/skills/workspace-studio-node-addon
curl -fsSL https://raw.githubusercontent.com/vinay-google/agent-skills/main/skills/workspace-studio-node-addon/SKILL.md \
  -o .agents/skills/workspace-studio-node-addon/SKILL.md
```
