# likho-docs

The documentation site of the Likho platform. It is a [Logseq](https://logseq.com) graph:
plain Markdown pages in `pages/`, published as a website by a GitHub Action on every push
to `main` (the same setup as docs.torahanytime.com).

## Read

After the first push and with GitHub Pages set to the `gh-pages` branch, the site is at
`https://likho-ai.github.io/likho-docs/` and opens on the Introduction page.

## Edit

* In Logseq: **Add a graph** and choose this folder. Edit pages; commit and push.
* In any editor: pages are outlines. Every block starts with `- `; a block's children are
  indented with one tab; continuation lines of a block (tables, code) are indented two spaces
  past the dash. Link a page with `[[Page Name]]`.
* A file name is the page name. `adr___0001 Separate repositories.md` is the page
  `adr/0001 Separate repositories` (three underscores make a namespace).

## Pages

| Page | Content |
| --- | --- |
| Introduction | What Likho is, what to read next, the repositories, the status |
| User guide | Every screen, for the people who read, correct and audit calls |
| Setup and start | From nothing to the first transcribed call on one machine |
| Configuration | The .env files, the settings that matter, the workspace settings |
| Operations | Health, metrics, the dashboard, stuck jobs, backups, troubleshooting |
| Services | Each service: stack, folders, ports, data, gRPC calls, events |
| Data model | Where every kind of data lives, and the main tables |
| Platform | Event bus, sign-in, Kubernetes, gateway, Google Cloud, CI/CD |
| Frontend | Micro-frontends: the shell, the apps, the shared packages |
| UI Design | Colours in light and dark, type, components, every screen |
| API Documentation | REST, GraphQL, gRPC and events, as built |
| Glossary | The words used across Likho |
| Roadmap | What is done, what comes next, step by step |
| Research | Open-source projects and speech models that were studied |
| adr/… | Decision records |

## What does not belong here

This repository is public. Anything about a customer's systems, data, call volumes, people
or credentials stays out of it.

## Publishing

Every push to `main` publishes the site (GitHub Pages, branch `gh-pages`). If a page does not
appear, check `logseq/config.edn` (`:publishing/all-pages-public? true`).
