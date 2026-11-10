# AGENTS.md: Learning Library

A personal, long-running library of deep-dive **entries** on programming problems and questions the owner hit. Each entry teaches one problem thoroughly enough to learn from months later. The owner is a junior C#/ASP.NET developer who wants the *why* behind every decision, not just the answer.

This repo is **public**. Everything committed is visible to the world.

## Layout

- `README.md`: the main index (תוכן עניינים). Every entry is listed here.
- `questions/<topic>/<entry-slug>.md`: all entries live under `questions/`, one entry per file, grouped by topic folder.
- `_templates/entry-template.md`: the template every entry follows. Read it before writing an entry.
- `sources/README.md`: the list of source files available locally. The files themselves are local-only and untracked.

## Adding an entry

The owner gives you a problem or question, usually in a sentence or two. You write the full entry; the owner reviews it.

1. **Generalize.** Restate the problem without anything that identifies the owner's employer or its systems (see *Public-repo safety*). Done when the problem statement would make sense to any developer.
2. **Check for overlap.** Search existing entries (`README.md` and `grep`). If an entry already covers this problem, extend that entry instead of creating a new one. Done when you have either a new slug or a target entry to extend.
3. **Pick topic and slug.** Choose the primary topic from *Topics*. Slug is lowercase kebab-case describing the problem, e.g. `stale-cache-after-ambiguous-db-write`. Done when the path `questions/<topic>/<slug>.md` is decided and free.
4. **Research.** Follow *Research and citations*. Done when every factual claim you plan to make has a source, or you have marked it unverified.
5. **Write.** Copy `_templates/entry-template.md` and fill every section. Set `status: draft`. Done when all ten sections have real content and no template guidance comments remain.
6. **Link.** Add the entry to the right topic section of `README.md` and to the top of *Recent additions* (keep the 10 newest). Add reciprocal links in the *Related* section of any entry you linked to. Done when every link in both directions resolves to an existing file.
7. **Safety check.** Run the *Public-repo safety* check over the diff. Done when it is clean.
8. **Hand to the owner for review.** Commit only when the owner asks; push only when the owner asks.

When the owner reviews an entry and approves it, set `status: reviewed` and update `updated:`.

## Entry rules

- **One entry per problem or question.** An entry pulls in whatever concepts it needs. Split a concept into its own entry only when a second entry needs it too; then both link to it.
- **Depth over breadth.** Explain the mechanism (what actually happens, step by step, in time order) before the fix. A timeline or Mermaid diagram usually helps.
- **Options, not a single answer.** Give at least 3 realistic options. Each gets a grade from 1 to 10, pros, cons, and when to use it, tied to a realistic C#/ASP.NET app. The grade scale is defined in the template. End the section with a grade table and a recommendation, including what it costs.
- **Name the concepts.** Use the industry name of every pattern and anti-pattern, so the owner can search for it.
- **English** throughout. The only Hebrew is the index heading.
- **Code examples** are C# on current LTS .NET with ASP.NET Core, and state the .NET version. Use pseudocode only when the idea is language-independent. Keep examples short, compilable in spirit, and focused on the concept.

## Topics

Topic folders live under `questions/` and are created only when the first entry needs them. Use an existing topic whenever one fits; an entry has one primary folder and can list more in its `topics:` metadata.

`databases`, `caching`, `distributed-systems`, `aspnet-core`, `csharp`, `architecture`, `testing`, `devops`

To add a topic, add it to this list and add its section to `README.md` in the same change.

## Research and citations

- **Primary sources first:** official docs (Microsoft Learn, vendor docs), papers, specifications, and well-known books. Blog posts are acceptable for community practice; label them as such.
- **Cite inline** with numbered markers like `[1]` that map to the entry's *Sources* list. Each source has a title, author or publisher, and a link (or chapter and section for books).
- **Unverified claims** are marked inline with **[unverified]** and listed in a short note at the end of *Sources*. Mark anything you could not confirm from a source you actually read.
- **Paraphrase** books and docs in your own words. Quote at most one short sentence per source, in quotation marks.
- **Local sources:** check `sources/README.md` for files the owner has on disk. Read them when relevant; they are optional.
- **Cite-only sources** (listed in `sources/README.md` with no local file) are cited by chapter and section from the owner's own copy or from what you can verify; their files are never added to the repo.

## Public-repo safety

Before every commit, check the diff for:

- **Employer specifics:** company, product, client, team, or internal service names; internal URLs; real table, queue, or endpoint names. Replace with generic ones (`OrdersService`, `orders` table).
- **Secrets:** connection strings, keys, tokens, passwords, hostnames, IPs.
- **Source files:** only `sources/README.md` is tracked under `sources/`. `.gitignore` enforces this; keep it that way.

Commits use the repo's local identity (`omrishe` with the GitHub noreply email), already configured in `.git/config`.

## Editing existing entries

Update `updated:` in the metadata on any meaningful change. Keep the slug stable once an entry exists, since other entries link to it; if a rename is unavoidable, fix every inbound link in the same change.
