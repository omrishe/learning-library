---
title: Home
nav_order: 1
permalink: /
---

# Learning Library: Table Of Contents

Deep-dive notes on problems and questions I hit while programming, mostly in C# and ASP.NET Core. Each entry explains why the problem happens, the realistic options and their tradeoffs, what the industry does, and where to read more.

Browse it as a website: **[omrishe.github.io/learning-library](https://omrishe.github.io/learning-library/)**, with a sidebar and search.

Rules for adding entries are in [AGENTS.md](https://github.com/omrishe/learning-library/blob/main/AGENTS.md). Every entry follows [the entry template](https://github.com/omrishe/learning-library/blob/main/_templates/entry-template.md).

**Status:** `draft` means written but not yet reviewed by me. `reviewed` means I've read and checked it.

## Recent additions

- 2026-11-12: [Changing a contract in an established project without breaking consumers](questions/architecture/changing-contracts-in-an-established-project.md)
- 2026-11-12: [Keeping a project's decisions and conventions findable by AI assistants and new developers](questions/best-practices/documenting-project-decisions-for-ai-and-new-developers.md)
- 2026-11-11: [ACID: what a database transaction actually guarantees](questions/best-practices/principles/acid-transactions.md)
- 2026-11-11: [SOLID: what the five principles mean and when they help or hurt](questions/best-practices/principles/solid-principles.md)
- 2026-11-10: [Passing IDs between the frontend and the backend](questions/aspnet-core/passing-ids-between-frontend-and-backend.md)
- 2026-11-10: [Passing IDs between services](questions/distributed-systems/passing-ids-between-services.md)
- 2026-11-10: [Error handling that depends on message text instead of error codes](questions/architecture/error-handling-by-message-text.md)
- 2026-11-10: [Stale cache reads while a DB write's outcome is unknown](questions/caching/stale-cache-after-ambiguous-db-write.md)

## Topics

### Caching

- [Stale cache reads while a DB write's outcome is unknown](questions/caching/stale-cache-after-ambiguous-db-write.md): why a cache serves old data during and after a timed-out write, and the patterns that bound it. *intermediate · draft*

### Databases

*No entries yet.*

### Distributed systems

- [Passing IDs between services](questions/distributed-systems/passing-ids-between-services.md): entity IDs, correlation (trace) IDs and message IDs, and how each should travel between services. *intermediate · draft*

### ASP.NET Core

- [Passing IDs between the frontend and the backend](questions/aspnet-core/passing-ids-between-frontend-and-backend.md): JSON precision for 64-bit IDs, where IDs go in a request, IDOR/BOLA, and int vs GUID vs public IDs. *intermediate · draft*

### C#

*No entries yet.*

### Architecture

- [Changing a contract in an established project without breaking consumers](questions/architecture/changing-contracts-in-an-established-project.md): why a rename that compiles can still break deployed consumers, and five ways to change a contract (additive-only, expand/migrate/contract, versioning, adapters, lockstep release). *intermediate · draft*
- [Error handling that depends on message text instead of error codes](questions/architecture/error-handling-by-message-text.md): why errors created as sentences break translation, UI logic and stored data, and how stable error codes with Problem Details fix it. *intermediate · draft*

### Testing

*No entries yet.*

### DevOps

*No entries yet.*

### Best practices

- [Keeping a project's decisions and conventions findable by AI assistants and new developers](questions/best-practices/documenting-project-decisions-for-ai-and-new-developers.md): seven ways to record conventions and the reasons behind decisions (AGENTS.md, ADRs, scoped rules, specs, docs-as-code, tooling), and why a short hand-written map plus ADRs plus automated checks is the usual starting point. *intermediate · draft*

#### Principles

- [SOLID: what the five principles mean and when they help or hurt](questions/best-practices/principles/solid-principles.md): what each of SRP, OCP, LSP, ISP and DIP protects against, where ASP.NET Core already applies them, and why applying all five everywhere up front backfires. *intermediate · draft*
- [ACID: what a database transaction actually guarantees](questions/best-practices/principles/acid-transactions.md): what atomicity, consistency, isolation and durability each promise, why the default READ COMMITTED level still loses updates, and how to keep concurrent writes correct with EF Core. *intermediate · draft*
