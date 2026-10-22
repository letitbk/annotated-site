# annotated-site

One skill that builds a small website where collaborators can comment on your markdown documents.

## The problem

Collaborators who do not use Git still need to review plans, codebooks and drafts. Emailing files loses the thread, and shared documents drift away from the version in the repository.

## What it does

The skill scaffolds a Next.js site, hosted on Vercel and protected by a password, for one or more markdown files in your project. Readers select text and leave comments, reply to each other, mark comments resolved and leave an overall comment per document. A script downloads all comments back into the repository as markdown, so they are tracked in Git next to the documents.

## Install

Claude Code, as a plugin:

```bash
claude plugin marketplace add letitbk/annotated-site
claude plugin install annotated-site@annotated-site --scope user
```

Codex and other agents that read Agent Skills:

```bash
npx skills add letitbk/annotated-site -g -a codex
```

Replace `codex` with `claude-code` to install the plain skill folders for Claude Code instead of the plugin. You can also copy the folders under `skills/` into `~/.claude/skills/` or `~/.codex/skills/` by hand.

This repository holds `annotated-site`. Start a new agent session after installing, then ask for the task in ordinary words or name the skill.

## Requirements

- Node.js and a Vercel account with the Vercel CLI signed in.
- A Neon Postgres database, added through the Vercel integration. The skill walks you through it.

## Limits

- The site uses one shared password, not individual accounts. Do not put confidential material on it.
- Each run generates about 1,100 lines of code tailored to your project. Review it like any other code before you deploy.

## License

MIT. See `LICENSE`.
