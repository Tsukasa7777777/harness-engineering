# Contributing to Harness Engineering

Thank you for your interest in contributing to Harness Engineering! This document provides guidelines and standards for contributions.

## How to Contribute

### Reporting Issues

- Use GitHub Issues to report bugs or suggest enhancements
- Provide clear, actionable descriptions
- Include examples or references where possible

### Submitting Changes

1. **Fork** the repository
2. **Create a branch** from `main` with a descriptive name:
   - `feat/add-eval-metric` for new features
   - `fix/guardrail-typo` for fixes
   - `docs/update-readme` for documentation
3. **Make your changes** following the style guidelines below
4. **Submit a Pull Request** with a clear description of what and why

## Style Guidelines

### Markdown Files

- Use ATX-style headers (`#`, `##`, `###`)
- Use fenced code blocks with language identifiers
- Keep lines under 120 characters where practical
- Use tables for structured data
- Include a YAML frontmatter block where appropriate

### File Naming

- Use `kebab-case` for file names: `agent-identity.md`, `eval-framework.md`
- Use descriptive names that indicate content, not just category

### Directory Structure

- Each section directory (`01_context/`, etc.) should have a `README.md` explaining its purpose
- Use subdirectories for grouping related files when a section grows beyond 5-7 files
- Remove `.gitkeep` files when adding real content to previously empty directories

### Content Standards

- **Be specific** — Vague guidelines are worse than no guidelines
- **Include examples** — Show, don't just tell
- **Reference sources** — Link to research, docs, or prior art
- **Version your specs** — Use semantic versioning headers in specification documents

## Commit Messages

Follow conventional commits:

```
feat: add evaluation framework for tool-use accuracy
fix: correct escalation threshold in guardrails
docs: expand agent identity template with examples
refactor: reorganize workflow patterns by complexity
```

## Code of Conduct

- Be respectful and constructive
- Focus on the work, not the person
- Assume good intent
- Welcome newcomers

## Questions?

Open a GitHub Issue or start a Discussion. We are happy to help!
