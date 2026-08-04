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

### salesforce-pss-icm

Reference and demo-build guide for Salesforce Public Sector Solutions (PSS) Investigative Case Management (ICM). Covers courts, tribunals, complaint intake, case proceedings, evidence and custody chain, regulatory violations, care plans, and citizen portals on Case / PublicComplaint / CaseProceeding / CustodyItem / RegulatoryCodeViolation.

Bundled reference covers the full 24-object data model, typical baseline OmniStudio inventory, DR / IP / OmniScript / FlexCard patterns specific to ICM, permission set group architecture, record types for courts vs tribunals, and OmniScript on LWR Experience Cloud gotchas.

## License

MIT — see `LICENSE`.
