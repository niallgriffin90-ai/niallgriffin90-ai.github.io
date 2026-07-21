---
title: "Session brief — Deployment and domain connection"
date: 2026-06-23
tags: [blog, hugo, github, dns, session-brief]
project: griffinai-blog
---

# Session brief — 23 June 2026

Covers Phase 6 (making the site live via automated deployment) and Phase 7
(connecting the custom domain). Written to be re-read cold: each section states
what was done, why, and what it means.

---

## Starting point

The site existed as source files in a GitHub repository
(`niallgriffin90-ai/niallgriffin90-ai.github.io`) but was not yet being served
on the internet. The goal today was to make it build and publish automatically,
then attach the real domain `griffinai.dev`.

---

## Phase 6 — Automated deployment (CI/CD)

### What was set up

A GitHub Actions *workflow file* was added at `.github/workflows/hugo.yaml`.

### What a workflow is

GitHub Actions is an automation system built into GitHub. A workflow is a set of
instructions, written in YAML, that GitHub runs on its own servers when a trigger
occurs. The trigger here is "a push to the `main` branch." When triggered, the
workflow spins up a temporary Linux machine, installs Hugo on it, builds the site,
and publishes the result to GitHub Pages.

This is what "CI/CD" means in practice — Continuous Integration / Continuous
Deployment. The effect: once set up, publishing a new post is just `git push`.
No manual building, no manual uploading. The pipeline does it.

### The critical line for this setup

```yaml
with:
  submodules: recursive
```

The PaperMod theme is included in the repo as a *git submodule* — a pointer to
another repository rather than a copy of its files. Without `submodules: recursive`,
GitHub clones the project but not the theme it points to, and the build fails with
a "theme not found" error. This line tells GitHub to pull the theme in too.

### The baseURL change

`hugo.yaml` line 1 was changed from the placeholder `https://example.org/` to
`https://griffinai.dev/`. This is the canonical address Hugo uses when generating
internal links. Note: the deployment workflow temporarily overrides this at build
time (via a `--baseURL` flag) so the site also works correctly at the
`niallgriffin90-ai.github.io` address before the custom domain is connected. Both
can coexist; this is expected, not a conflict.

### The GitHub Pages source setting

In the repo: Settings → Pages → Build and deployment → Source was changed from
"Deploy from a branch" to "GitHub Actions." This tells GitHub that the workflow
file is now responsible for deployment, rather than GitHub trying to serve files
directly from a branch.

### Result

After pushing, the workflow ran (visible under the repo's Actions tab) and
deployed. The site became reachable at `https://niallgriffin90-ai.github.io`.

---

## Problems encountered in Phase 6 (and what they taught)

These are worth keeping because the *reasons* generalise.

### 1. Commands ran in the wrong directory

The workflow file was first created while the terminal was sitting in the home
directory (`~`), not the project folder. The files landed in `~/.github/` instead
of inside the project.

**Why it happened:** in VS Code, the folder shown in the sidebar and the folder
the integrated terminal is "in" are independent. Editing a file in the sidebar
does not move the terminal. The terminal's location is shown in its prompt — here,
`~$` means home, `~/Downloads/niallgriffin$` means inside the project.

**The lesson:** always check the prompt before running file-creating commands.
`pwd` prints the current directory; `cd ~/Downloads/niallgriffin` moves into the
project.

### 2. Push rejected — token missing `workflow` scope

The first push of the workflow file was rejected with:
`refusing to allow a Personal Access Token to create or update workflow ...
without 'workflow' scope`.

**Why it happened:** GitHub treats files inside `.github/workflows/` as sensitive,
because a workflow can run arbitrary code on GitHub's servers. A Personal Access
Token needs an explicit, separate permission — the `workflow` scope — to push such
files. The token had `repo` but not `workflow`.

**The fix:** edit the existing token on GitHub (Settings → Developer settings →
Personal access tokens → Tokens classic → select the token → tick `workflow` →
Update token). The token value does not change, so nothing else needed updating.

### 3. A note file got committed into the repo

The session-explainer note was moved into the project folder and was swept into a
commit.

**Why it matters:** anything inside the project folder is part of the git
repository and will be pushed to the public GitHub repo — even if it does not
appear on the website. (The website only publishes what is inside `content/`; the
repo contains everything.) So a note placed anywhere in the project is heading for
the public repo.

**The rule going forward:** notes and drafts live entirely outside the project
folder. They are now kept in `~/blog-notes/`.

### 4. Cleaning the bad commit — `git reset --soft`

Because the commit containing the note had not been successfully pushed (it was
blocked by the token error), it existed only locally and could be safely rewritten.

```bash
git reset --soft HEAD~1   # undo the last commit, keep all file changes
git add -A                # re-stage the current state (note now removed)
git commit -m "..."       # make a fresh commit without the note
```

`reset --soft HEAD~1` rewinds one commit but leaves the working files untouched.
Re-staging then captures the current state, which no longer includes the note.
The result is a commit whose history never contained the note. This is only safe
for commits that have **not** been pushed; rewriting already-pushed history causes
problems for anything that has pulled it.

### 5. The `~/Documents` symlink loop

Attempts to use `~/Documents/niallgriffin-notes` failed with
`Too many levels of symbolic links` (a filesystem loop, ELOOP). The cause was not
determined. It is unrelated to the blog. It was sidestepped entirely by using
`~/blog-notes/` instead. The `~/Documents` issue remains open and can be
investigated separately if needed.

---

## Phase 7 — Connecting the domain `griffinai.dev`

Connecting a domain has two halves that must both be done: tell GitHub to accept
the domain, and tell the registrar (Namecheap) where to send visitors.

### Half 1 — GitHub side

Repo → Settings → Pages → Custom domain → entered `griffinai.dev` → Save.

This makes GitHub willing to serve the site under that name, and it creates a
`CNAME` file in the repo so the setting persists across rebuilds.

Decision made: the *apex* (bare) domain `griffinai.dev` is the primary, rather
than `www.griffinai.dev`. The apex is cleaner for a CV/LinkedIn. The `www` version
redirects to it.

### Half 2 — Namecheap DNS records

In Namecheap → Domain List → Manage → Advanced DNS → Host Records.

**What DNS is:** the Domain Name System is the internet's address book. It
translates a human-readable name (`griffinai.dev`) into the numerical server
addresses computers use. A *record* is one entry in that address book.

The following five records were added:

| Type  | Host | Value                          | Meaning |
|-------|------|--------------------------------|---------|
| A     | @    | 185.199.108.153                | Apex → GitHub server 1 |
| A     | @    | 185.199.109.153                | Apex → GitHub server 2 |
| A     | @    | 185.199.110.153                | Apex → GitHub server 3 |
| A     | @    | 185.199.111.153                | Apex → GitHub server 4 |
| CNAME | www  | niallgriffin90-ai.github.io.   | www → the GitHub Pages host |

Explanations:

- **A record** maps a name directly to an IP address. `@` is Namecheap's notation
  for the bare domain itself (`griffinai.dev` with no prefix). Four A records is
  correct — GitHub Pages publishes four server addresses and traffic is spread
  across them for reliability. The four IPs are GitHub's standard Pages addresses,
  identical for every GitHub Pages user.
- **CNAME record** maps a name to *another name* rather than an IP. The `www` host
  is pointed at `niallgriffin90-ai.github.io`, so `www.griffinai.dev` resolves to
  the GitHub Pages host. The trailing dot is standard DNS notation for a fully
  qualified name.

### Records that were removed

The Namecheap defaults included a `www` CNAME and a URL Redirect record. Both were
deleted, because they would conflict with the records above. (The old www CNAME was
replaced with the new one pointing at GitHub.)

### A record that was left in place

A `TXT` record with value beginning `v=spf1 include:spf.efwd.re...` was left
untouched. This is an **SPF record** — it concerns *email*, not the website. SPF
(Sender Policy Framework) lists which servers are permitted to send email claiming
to be from the domain; it is an anti-spoofing measure. It does not interact with
the A or CNAME records (web traffic and email are governed separately), and it
would be needed if email forwarding on the domain is set up later. No reason to
remove it.

---

## Current state at end of session

- Site is live at `https://niallgriffin90-ai.github.io`.
- DNS records for `griffinai.dev` are configured at Namecheap.
- `griffinai.dev` is registered as the custom domain in GitHub Pages settings.
- DNS propagation is in progress — changes ripple across the internet's servers
  and can take from ~30 minutes up to 24 hours. Nothing to do but wait.

## Outstanding — to do once propagation completes

1. Confirm `https://griffinai.dev` loads the site.
2. In GitHub repo → Settings → Pages, tick **Enforce HTTPS**. This may be greyed
   out for up to an hour after DNS resolves, while GitHub provisions a free
   security certificate (via Let's Encrypt). The `.dev` TLD requires HTTPS, so
   this step is necessary, not optional.

## Open item (unrelated to blog)

- `~/Documents` has a symbolic-link loop causing `Too many levels of symbolic
  links`. Cause unknown. Parked. Notes are kept in `~/blog-notes/` to avoid it.

---

## Glossary of terms used today

- **CI/CD** — Continuous Integration / Continuous Deployment. Automation that
  builds and publishes a project automatically on each change.
- **GitHub Actions** — GitHub's built-in automation system; runs workflows.
- **Workflow** — a YAML instruction file defining what automation runs and when.
- **Submodule** — a git repository referenced inside another repository as a
  pointer, not a copy.
- **baseURL** — the canonical web address Hugo uses to build links.
- **Personal Access Token (PAT)** — a generated credential for authenticating
  with GitHub; carries specific permission *scopes*.
- **Scope** — a specific permission attached to a token (e.g. `repo`, `workflow`).
- **DNS** — Domain Name System; translates names into IP addresses.
- **A record** — maps a name to an IP address.
- **CNAME record** — maps a name to another name.
- **Apex / bare domain** — the domain with no prefix (`griffinai.dev`).
- **SPF record** — a TXT record listing servers allowed to send email for a
  domain; anti-spoofing; unrelated to web hosting.
- **Propagation** — the delay while DNS changes spread across internet servers.
- **HTTPS / Let's Encrypt** — encrypted web connection; Let's Encrypt is the free
  certificate authority GitHub uses to provide it.
