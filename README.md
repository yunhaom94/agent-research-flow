# Agent Research Flow

A Claude plugin and workspace for automating academic research workflows.

## Capabilities
- **Structured Idea Development**: Develop research ideas in a structured format (`develop-idea`).
- **Search & Download**: Automated literature discovery (`literature-search`).
- **Manage & Ideate**: Zotero integration  (`zotero-manage`). (Not yet implemented)

## Quick Start

Use `develop-idea` skill to initalize a project workspace. The process revolves around `references/Ideas.md` as the main workspace, where human and the AI agent collabrativly build and expand a research idea.

## Load As Claude Code Plugin

To load this repository as a local plugin in Claude Code, run:

```bash
git clone <this-repo-url>
claude --plugin <path-to-this-directory>
```

You need to set up the API key for Semantic Scholar search as an environment variable. For example, under `<project_root>/.claude/settings.local.json`:
```{
  "env": {
    "S2_API_KEY": <your-semantic-scholar-api-key>
  },
}
```
