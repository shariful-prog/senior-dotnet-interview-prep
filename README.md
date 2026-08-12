# Senior .NET Interview Prep

Interview notes for **C#**, **.NET / ASP.NET Core**, **SQL**, **JavaScript**, and **Architecture**, published as a static site with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

Each topic comes in two depths:

- **Top 20** — a one-line definition, then the detail. For the day before an interview.
- **Deep dive** — full sections with code examples, gotchas, and a cheat sheet at the end.

## Running locally

```bash
pip install -r requirements.txt
mkdocs serve
```

Then open <http://127.0.0.1:8000>.

The dev server watches `docs/` and `mkdocs.yml` and reloads on save.

## Building

```bash
mkdocs build --strict
```

Output goes to `site/` (git-ignored). `--strict` turns broken internal links and
bad anchors into build failures — this is what CI runs, so build locally with it
before pushing.

## Publishing

Pushing to `main` triggers [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml),
which builds the site and deploys it to GitHub Pages.

One-time setup: in the repository, go to **Settings → Pages** and set
**Source** to **GitHub Actions**.

## Layout

```
docs/
├── index.md           landing page
├── Top20/             quick revision — 5 topics × 20 questions
├── C-Sharp/           type system, OOP, generics, LINQ, async, memory …
├── Dotnet/            DI, ASP.NET Core, Web APIs, EF Core, security, ops
├── Sql/               joins, indexing, transactions, schema design, problems
├── JS/                JavaScript, TypeScript, Angular, code-output practice
├── Architecture/      SOLID, patterns, Clean Architecture, microservices
├── Senerios/          scenario questions (in the repo, not published)
└── assets/extra.css   theme customization
```

## Conventions

Worth knowing before editing:

**Heading levels** are consistent across every file:

```markdown
#     file title
##    section        (B1 — OOP, O3 — Loading Strategies)
###   question       (Q1., Q2., P1. …)
####  sub-heading inside an answer
```

**Each question ends with a `---` separator.** The theme adds no rule of its own,
so the separator in the Markdown is the only one — don't add a second.

**Filenames are topic-descriptive** (`csharp-async.md`, `dotnet-ef-core.md`).
Section headings inside the files still carry their letter codes (`## B1 —`,
`## O3 —`) because the anchors are what cross-file links point at.

**Anchors are GitHub-compatible.** `toc.slugify` in `mkdocs.yml` is set to
GitHub's algorithm, so a link like `csharp-type-system.md#a2--boxing--unboxing`
resolves both when browsing the Markdown on GitHub *and* on the built site.
Note the double hyphen — it comes from the em dash in the heading.

**Not published:** `Senerios/` and `Top20/index.md` are listed under
`exclude_docs` in `mkdocs.yml`. They stay in the repo but are left out of the
build. To publish them, remove the entry and add them back to `nav`.
