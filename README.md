# SAP Hybris Commerce Best Practices

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

A documentation-first skill bundle for SAP Commerce Cloud (Hybris) development best practices.
It is intended to be installed as a skill for agentic coding assistants, and also works as a standalone reference.

## What is included

- Version context and entry point: `sap-hybris-commerce-best-practices/SKILL.md`
- Numbered guides in `sap-hybris-commerce-best-practices/references/` covering:
  - Code conventions and Java guidelines
  - Extensions, items.xml data modeling, ImpEx
  - Services, events, interceptors, validation
  - Testing and code quality
  - Facades, controllers, WCMS, JSP tags/views
  - Solr integration, tasks/cronjobs
  - Backoffice (cockpitng) configuration

Start here: `sap-hybris-commerce-best-practices/SKILL.md`

## How to use (skills.sh)

Install this repository as a skill with the `skills` CLI:

```bash
npx skills add sina96/sap-hybris-commerce-best-practices-skill
```

Then, in your agent/tooling, select or load the installed skill and use `SKILL.md` as the entry point for navigation.
When editing this repo itself, follow `AGENTS.md` (commands + style rules).

Note: for compatibility with tooling that expects a root `SKILL.md`, this repo includes a small redirect file at `SKILL.md`.

## License

MIT - see `LICENSE`.
