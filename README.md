# Claude DevOps Skills

A collection of Claude Skills that I will create for myself for the usecases of DevOps, infrastructure engineering, cloud platforms, CI/CD, Kubernetes, and infrastructure as code.

## Available Skills

| Skill | Description | Status |
|---|---|---|
| `pulumi-csharp-best-practices` | Best practices for writing, reviewing, and debugging Pulumi C# programs, including `Output<T>`, `Apply`, `ComponentResource`, aliases, secrets, preview workflows, Helm, and Kubernetes gotchas. | Ready |

## Repository Structure

```text
skills/
  <skill-name>/
    SKILL.md
    references/
    scripts/
    assets/

dist/
  <skill-name>.zip

```
skills/ contains the editable source for each skill.

dist/ contains upload-ready ZIP files for Claude.ai. Each ZIP must contain a top-level folder whose name matches the name field in SKILL.md.

## Installing a Skill in Claude.ai
Download the ZIP file from dist/ or from the GitHub Releases page.

Open Claude.ai.
Go to Customize.
Click + icon, Upload Skill.
Select the ZIP file.
Start a new chat and ask Claude to use the skill (or it will auto-invoke the skill when it sees a relevant conversation)
