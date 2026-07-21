---
title: "Session brief — Site setup and first push"
date: 2026-06-16
tags: [blog, hugo, git, github, session-brief]
project: griffinai-blog
phase: "1-5"
---

# Session brief — 16 June 2026

Scope: installing the toolchain, scaffolding the Hugo site, configuring it,
adding the legal pages and post template, and pushing the source to GitHub under
a separate identity. Covers Phases 1–5 of the build.

---

## Hugo Extended

Hugo is a static site generator. Posts are written as plain Markdown files; Hugo
converts them into a complete website (HTML and CSS) that any browser can serve.

Hugo ships in two editions, standard and extended. The extended edition processes
SCSS/Sass stylesheets, which most themes — including PaperMod — require. The
standard edition fails on those themes with an SCSS error. Extended was installed.

Reason for choosing Hugo over a hosted platform (WordPress, Squarespace): the
output is just static files. No database, no always-on server, nothing to patch
or that can go down. It is fast, free to host, and the entire site lives in Git,
which makes it version-controlled and portable.

---

## Git and GitHub

These are distinct things.

Git is software running locally that tracks changes to files over time. Each
`git commit` records a snapshot of the project. Any snapshot can be returned to,
and the difference between any two snapshots can be inspected. It functions as a
complete change history for the project.

GitHub is a website that stores copies of Git repositories remotely. Git is the
system; GitHub is one host for it (GitLab, Bitbucket, or a private server are
alternatives — all use Git). GitHub was chosen because of GitHub Pages, its free
static-site hosting, used in a later phase.

Actions taken: `git init` created a repository inside the project folder; the
first commit recorded the initial state; that commit was pushed to a new GitHub
repository under a separate account.

---

## Site structure

`hugo new site` scaffolds a fixed folder layout. Function of each:

- `content/` — written content. Each `.md` file becomes a page. Created here:
  `about.md`, `privacy.md`, `disclaimer.md`, `archives.md`. Posts live in
  `content/posts/`.
- `themes/` — the visual design. PaperMod sits here. These files are not edited
  directly; the theme is controlled from the config file.
- `archetypes/` — templates for new content. `hugo new posts/x.md` uses the
  template here to pre-fill a new file.
- `static/` — files served as-is (images, favicon, the future `llms.txt`). Not
  processed by Hugo.
- `public/` — where Hugo writes the built site. Not edited directly; excluded
  from Git via `.gitignore` because the deployment pipeline regenerates it on
  each push.
- `hugo.yaml` — the configuration file. Controls site title, theme, menu, social
  icons, and SEO settings. The homepage text, the menu, and the social icons are
  all defined here.

---

## PaperMod theme

PaperMod is an open-source Hugo theme. It supplies the layout, typography,
light/dark toggle, reading-time indicator, and breadcrumb navigation. Only
content, identity, and configuration are user-supplied.

It was installed as a Git submodule — a reference inside the repository pointing
to another repository, rather than a copy of its files. Consequence: theme
updates can be pulled with one command, and the project repository stays small,
holding a pointer rather than the theme's full file tree.

---

## Frontmatter

Every Hugo content file opens with a metadata block delimited by `---`. This is
the frontmatter — data about the file rather than its content.

```yaml
---
title: "About"
layout: "single"
url: "/about/"
ShowReadingTime: false
---
```

Hugo reads it before rendering: `title` sets the page title, `layout` selects the
presentation, `url` sets the page address. Page content follows the second `---`.
The post archetype pre-fills this block so new posts start with the correct
structure.

---

## Identity separation — two GitHub accounts

Git carries two independent identities:

1. Commit identity — the name and email stamped on each commit, visible in public
   history.
2. Authentication identity — the GitHub account used to connect and push.

`git config user.name` set *without* `--global` sets the commit identity for this
project only, leaving the global (`groggs`) identity intact elsewhere. This was
done deliberately to keep the blog's commit history under the professional name.

Authentication is separate. VS Code's terminal used the system's stored
credentials (`groggs99`) when pushing, producing a 403 Permission Denied: GitHub
saw `groggs99` pushing to a repository owned by `niallgriffin90-ai` and refused.

Fix: a Personal Access Token (a generated credential scoped to specific
permissions) embedded in the remote URL —
`https://niallgriffin90-ai:TOKEN@github.com/...` — which bypasses the system
credential store entirely.

---

## Repository name convention

The repository is named `niallgriffin90-ai.github.io`. GitHub treats a repository
named exactly `<username>.github.io` as the account's personal GitHub Pages site
and serves it at that address. The name is functional, not cosmetic — it is what
enables the free hosting.

---

## State at end of session

- Toolchain installed (Hugo Extended, Git present).
- Site scaffolded, PaperMod added, config written.
- Pages created: About, Privacy, Disclaimer, Archive.
- Post archetype created (new posts default to `draft: true`).
- Source pushed to `niallgriffin90-ai/niallgriffin90-ai.github.io`.
- Commit identity confirmed as the professional account, not `groggs`.

## Decisions made

- Separate GitHub account (`niallgriffin90-ai`) for the blog, isolating it from
  the existing `groggs` crypto/Web3 identity.
- Per-repo Git identity rather than global, so only this project carries the new
  name.
- Notes and drafts kept outside the project folder (anything inside it is part of
  the public repo).

## Glossary

- Static site generator — software that builds a website from plain text files.
- Commit — a recorded snapshot of the project in Git.
- Submodule — a repository referenced inside another as a pointer, not a copy.
- Frontmatter — the metadata block at the top of a content file.
- Personal Access Token (PAT) — a generated GitHub credential with defined scopes.
- GitHub Pages — GitHub's free static-site hosting.
