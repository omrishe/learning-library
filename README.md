# Learning Library: Table Of Contents

Deep-dive notes on problems and questions I hit while programming, mostly in C# and ASP.NET Core. Each entry explains why the problem happens, the realistic options and their tradeoffs, what the industry does, and where to read more.

Rules for adding entries are in [AGENTS.md](AGENTS.md). Every entry follows [the entry template](_templates/entry-template.md).

**Status:** `draft` means written but not yet reviewed by me. `reviewed` means I've read and checked it.

## Recent additions

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

*No entries yet.*

### C#

*No entries yet.*

### Architecture

- [Error handling that depends on message text instead of error codes](questions/architecture/error-handling-by-message-text.md): why errors created as sentences break translation, UI logic and stored data, and how stable error codes with Problem Details fix it. *intermediate · draft*

### Testing

*No entries yet.*

### DevOps

*No entries yet.*
