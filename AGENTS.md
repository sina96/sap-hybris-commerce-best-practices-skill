# Repository Agent Guide (AGENTS.md)

This repo is a documentation/skill bundle for SAP Commerce Cloud (Hybris) best practices.
Most changes are edits to Markdown in `sap-hybris-commerce-best-practices/SKILL.md` and
`sap-hybris-commerce-best-practices/references/*.md`.

## Repo Layout

- `sap-hybris-commerce-best-practices/SKILL.md`: main entry point; links to references.
- `sap-hybris-commerce-best-practices/references/NN-*.md`: numbered topic guides (keep numbering stable).
- `opencode.jsonc`: OpenCode per-project defaults (agent selection).
- `.opencode/`: local OpenCode runtime files (often ignored); avoid editing unless asked.

## Build / Lint / Test Commands

There is no project build (no `package.json`, `pom.xml`, `pyproject.toml`, etc.) at repo root.
Validation is primarily manual (Markdown quality + consistency).

### Repo Sanity Checks (this repo)

```bash
# Basic hygiene
git status
git diff

# Search across docs
rg "pattern" sap-hybris-commerce-best-practices

# Optional (only if you have the tools installed)
# - Markdown lint
markdownlint "**/*.md"
# - Link check
lychee --no-progress "**/*.md"
```

Notes:
- Do not invent scripts; if you add tooling, document it here and commit the config.
- Prefer small, focused doc changes; keep examples consistent with the conventions below.

### SAP Commerce (Hybris) Commands (run in a Commerce installation, not in this repo)

These are included because the docs reference them; they require a configured SAP Commerce home.

```bash
cd "$HYBRIS_HOME/bin/platform"
. ./setantenv.sh

# Build / generate
ant clean all
ant build -Dextension.name=mycompanycore
ant build -Dextension.name=mycompanycore -Dtarget=models

# Initialize / update
ant initialize
ant updatesystem

# Solr (if used)
ant startSolrServer
ant stopSolrServer

# Tests
ant alltests
ant unittests
ant integrationtests

# Run a narrower set of tests (example from docs)
ant integrationtests -Dtestclasses.extensions=mycompanycore

# Coverage (example from docs)
ant alltests -Dtestclasses.coverage=true
```

If you need true single-test-class execution, check your project-specific ant targets and
properties in the Commerce repo (they vary by version and setup).

## Code Style Guidelines (for docs and code snippets)

When editing this repo, you are usually editing guidance and examples. Keep examples aligned
with the conventions in `sap-hybris-commerce-best-practices/references/00-code-conventions.md` and
`sap-hybris-commerce-best-practices/references/01-java-guidelines.md`.

### Java Formatting

- Indentation: tabs (not spaces)
- Tab/indent size: 3
- Right margin: 130
- Braces: opening brace at end of line; `else/catch/finally/while` on a new line
- Wrapping: for long binary operations, put the operator on the next line

### Imports

- Avoid `*` imports unless the import-on-demand threshold is exceeded (> 20)
- Group and order imports, separated by blank lines:
  - `java.*`
  - `javax.*`
  - `org.*`
  - `com.*`
  - all others

### Architecture / Patterns (SAP Commerce)

- Service layer first: do not use Jalo directly (deprecated)
- Interface + implementation pattern:
  - `ProductService` + `DefaultProductService` (or similar)
  - Put implementations in `.impl` packages
- Dependency injection: constructor injection (no field injection)
- Transactions: use `@Transactional` to define boundaries; prefer `readOnly=true` for reads
- Do not modify generated model classes; put custom logic in services/handlers

### Naming Conventions

- Packages: `com.<company>.<extension>.<layer>` (examples in docs use `com.mycompany...`)
- Extensions: prefixed with company name (e.g., `mycompanycore`, `mycompanyfacades`)
- Classes:
  - Interfaces: nouns/roles (`OrderService`, `ProductDao`)
  - Implementations: `Default*` prefix is common in Commerce (`DefaultOrderService`)
  - DTOs: `*Data`; Populators: `*Populator`; Converters: `*Converter`
- Constants: `UPPER_SNAKE_CASE`
- items.xml:
  - Type codes and qualifiers are stable API; choose meaningful names and add descriptions
  - Prefer deprecating over removing fields in existing installations

### Types / Null Handling

- Validate public method inputs early; fail fast with clear messages
- Prefer `Optional` for "may be absent" returns; do not use `Optional` for fields
- Use `Objects.requireNonNull(...)` for required parameters

### Error Handling

- Use SAP Commerce exception hierarchy where appropriate:
  - `UnknownIdentifierException`, `AmbiguousIdentifierException`, `ModelNotFoundException`
- Do not swallow exceptions; wrap with context and preserve the cause
- Avoid catching raw `Exception` unless you rethrow a domain-specific exception
- Log with SLF4J; never use `System.out.println()` in examples

### Logging

- Levels:
  - `TRACE` for very detailed flow, `DEBUG` for debugging, `INFO` for key business events,
    `WARN` for suspicious conditions, `ERROR` for failures
- Messages: include identifiers (order code, product code) and keep PII out of logs

### Testing Guidance (examples)

- Unit tests: JUnit 5 + Mockito; follow AAA (Arrange/Act/Assert)
- Integration tests: extend `ServicelayerTest` and annotate with `@IntegrationTest`
- Test names: describe behavior and outcome (success + failure cases)
- Prefer builders/fixtures for test data; avoid flakiness (no `Thread.sleep()`)

## Markdown / Documentation Style

- Keep docs scannable: short sections, bullet lists, and concrete examples
- Use fenced code blocks with language tags (`java`, `xml`, `impex`, `bash`)
- Keep line lengths reasonable; avoid trailing whitespace
- Prefer ASCII characters unless the existing file already uses Unicode
- If you add a new topic doc:
  - follow the `sap-hybris-commerce-best-practices/references/NN-title.md` pattern
  - update `sap-hybris-commerce-best-practices/SKILL.md` topic list

## Cursor / Copilot Rules

- No `.cursorrules`, `.cursor/rules/`, or `.github/copilot-instructions.md` found in this repo.
