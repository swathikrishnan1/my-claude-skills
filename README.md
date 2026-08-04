# My Claude Code Skills

Personal skills for Salesforce work with Claude Code.

## Install

Clone into your Claude skills folder:

```bash
git clone https://github.com/swathikrishnan1/my-claude-skills.git ~/.claude/skills/my-claude-skills
```

Claude finds any `SKILL.md` recursively, so nested folders work fine.

## Skills

### omnistudio-retrieve-and-modify

A docs-first, retrieve-before-generate workflow for OmniStudio and BRE work in Salesforce PSS / Industries orgs.

Three layers:
1. Official Salesforce docs via the `salesforce-docs` MCP server
2. Bundled reference notes with undocumented real-build gotchas
3. Live-org retrieval via sf CLI + DataPacks

Bundles reference notes covering FlexCard → OmniScript ContextId plumbing, Apex Callable Remote Action wiring, activation gotchas, and BRE lookup-table structure.

## License

MIT — see `LICENSE`.
