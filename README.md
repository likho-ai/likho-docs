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
| Introduction | What Likho is, how it is built, the repositories |
| Services | Each service: stack, folders, ports, data, gRPC calls, events |
| Platform | Event bus, login (cookies, sessions, JWT), Kubernetes, gateway, Google Cloud, CI/CD |
| Frontend | Micro-frontends: the shell, the apps, the shared packages |
| UI Design | Colours in light and dark, type, components, every screen |
| API Documentation | REST, GraphQL, gRPC and events; Postman |
| Research | Open-source projects and speech models that were studied |
| adr/… | Decision records |

## What does not belong here

This repository is public. Anything about a customer's systems, data, call volumes, people
or credentials stays out of it.

## Not verified yet

The publish workflow has not run; it runs on the first push. If a page does not appear,
check `logseq/config.edn` (`:publishing/all-pages-public? true`).
