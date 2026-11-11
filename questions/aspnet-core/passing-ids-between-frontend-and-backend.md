---
title: Passing IDs between the frontend and the backend
parent: ASP.NET Core
topics: [aspnet-core, architecture]
tags: [json-number-precision, max-safe-integer, idor, bola, resource-based-authorization, uuid, uuidv7, public-id, enumeration-attack, parameter-binding]
difficulty: intermediate
status: draft
added: 2026-11-10
updated: 2026-11-10
---

# Passing IDs between the frontend and the backend

## TL;DR

An ID that crosses between the browser and your API runs into three separate problems:
- **Representation.** JavaScript stores every JSON number as a 64-bit float, so integer IDs above 2^53 − 1 are silently rounded. Send large IDs as strings.
- **Trust.** Every ID in a request is user input. The backend must check that the caller may access *that* object on every request. This check is the actual fix; unguessable IDs only make attacks harder.
- **Leakage.** Sequential IDs reveal how many records you have and let attackers walk through them. Random or separate public IDs avoid that.

A good default for a new ASP.NET Core app: GUID keys (or a separate public ID column), sent as JSON strings, in the route for "which resource", with an ownership check on every access.

## The problem

A single-page app talks to an ASP.NET Core API. Orders have a database ID, and the frontend needs it to show, edit and delete orders. The developer has several questions at once:

1. **What type should the ID be?** The table uses `bigint` IDs. One day a bug report arrives: opening an order gives "not found", and the ID in the request is **one digit off** from the real one. The backend sent `9007199254740993`; the browser asked for `9007199254740992`.
2. **Where should it go?** `GET /orders/42`, `GET /orders?id=42`, or `{ "id": 42 }` in a body?
3. **Is it safe to show it?** A user changes the URL from `/orders/42` to `/orders/43` and sees **another customer's order**. A competitor notices order IDs go up by about 300 a day and estimates your sales.

These look like one question ("how do I send an ID?") but they are three different mechanisms: number representation, trust, and information leakage.

## Why it happens

### 1. JavaScript numbers are doubles, so big integers get rounded

C# `long` is a 64-bit integer. JavaScript has no 64-bit integer type for ordinary numbers: every `number` is an IEEE 754 double, which can represent integers **exactly** only up to `Number.MAX_SAFE_INTEGER`, which is 2^53 − 1 = 9,007,199,254,740,991 [1]. Above that, different integers map to the same double. MDN's example: `MAX_SAFE_INTEGER + 1 === MAX_SAFE_INTEGER + 2` is `true` [1].

The JSON spec warns about exactly this. It only promises that implementations agree on integers in the range [-(2^53)+1, (2^53)-1] [3]. The backend writes the digits correctly; the browser's `JSON.parse` rounds them, with no error:

| Step | Value |
| --- | --- |
| C# `long Id` in the database | `9007199254740993` |
| JSON text sent by ASP.NET Core | `{"id":9007199254740993}` (correct) |
| After `JSON.parse` in the browser | `9007199254740992` (rounded, silently) |
| Next request | `GET /orders/9007199254740992` → 404, or a **different** order |

`BigInt` can hold the value, but JSON doesn't support it: `JSON.stringify` throws a `TypeError` on any `BigInt`, and `JSON.parse` only produces one with a custom reviver [2].

When does this bite? A database identity column rarely reaches 2^53. But IDs that pack a timestamp and machine bits into 64 bits ("Snowflake-style" IDs), hashes, or IDs from external systems can exceed it from day one. GUIDs never have the problem, because JSON already carries them as strings.

### 2. Every ID from the client is attacker-controlled

The browser is outside your trust boundary. Anyone can edit a URL or replay a request with a different ID. If the endpoint loads "order 43" without checking who owns it, that's **Insecure Direct Object Reference (IDOR)**, which OWASP's API Security Top 10 lists first as **Broken Object Level Authorization (BOLA)**, API1:2023 [7]. OWASP's fix is direct: run an access-control check for every object a user tries to access, every time [6][7].

`[Authorize]` alone can't do this check. In ASP.NET Core, attribute-based authorization runs before data binding and before the resource is loaded, so it can only answer "is this user logged in / in this role?", not "does this user own order 43?" [8]. That needs **resource-based authorization**: load the object, then check it.

### 3. Sequential IDs leak information

Auto-increment IDs are predictable. That causes two problems even when authorization is correct:
- **Enumeration.** An attacker can try 1, 2, 3, … and learn which records exist, which turns any authorization slip into a mass data leak.
- **Business metrics.** The gap between two IDs created a day apart tells outsiders your daily volume.

Random identifiers such as GUIDs make guessing practically impossible, and OWASP recommends them, but explicitly as **defense in depth**, not a replacement for the access check [6][7]. RFC 9562, the UUID standard, goes further: implementations "SHOULD NOT assume that UUIDs are hard to guess", and UUIDs must not be used as **security capabilities**, meaning IDs whose mere possession grants access [9]. Time-ordered UUIDv7 values contain a 48-bit Unix timestamp [9], so anyone holding one can read when the record was created.

### 4. Where the ID goes has rules too

ASP.NET Core minimal APIs bind parameters from the route, the query string, headers, or the JSON body. A simple-type parameter whose name appears in the route template binds from the route; otherwise it binds from the query string. `GET`, `HEAD`, `OPTIONS` and `DELETE` don't bind from the body implicitly [10]. The usual REST convention follows from this:
- **Route** for the ID that names the resource: `GET /orders/{id}`.
- **Query string** for filters and searches: `GET /orders?customerId=…`.
- **Body** for IDs that are part of the data being written: `{ "productId": "…" }`.

Two details trip people up. Route constraints like `{id:int}` or `{id:guid}` are for choosing between routes, not validation; Microsoft says "Don't use constraints for input validation", because a malformed ID then returns `404` instead of `400` [11]. And anything in a URL can end up in server logs, browser history and `Referer` headers [12], so an ID that is itself a **secret** (a password-reset token, an unlisted share link) needs more care than an ordinary record ID.

## Options and solutions

These options are about **which ID you expose and how**. Whatever you choose, the object-level authorization check from mechanism 2 is mandatory; it is not one of the options.

### Option 1: Expose the database integer as a JSON number

Send the `int` or `long` primary key as is: `{ "id": 42 }`, `/orders/42`.

- **Grade:** 5/10. Simplest, but it leaks counts, invites enumeration, and breaks for `long` values above 2^53.
- **Pros:** No extra work. IDs are short and easy to read in logs and support tickets. A 4- or 8-byte integer is the most compact key for the database.
- **Cons:** Enumerable and reveals volume (mechanism 3). A `long` that exceeds 2^53 − 1 is silently corrupted in the browser (mechanism 1). Every authorization slip is easy to exploit at scale.
- **Use when:** Internal admin tools, `int` keys, and data where counts aren't sensitive.

### Option 2: Database integer, sent as a JSON string

Keep integer keys, but serialize ID properties as strings: `{ "id": "9007199254740993" }`. System.Text.Json supports this with `JsonNumberHandling.WriteAsString`, per property with the `[JsonNumberHandling]` attribute or globally through `JsonSerializerOptions.NumberHandling` [4]. ASP.NET Core's web defaults already accept quoted numbers on input; writing them as strings is opt-in [4].

- **Grade:** 6/10. Fixes precision cheaply for existing `long` keys, but still enumerable.
- **Pros:** Small change, no schema migration. The frontend treats IDs as opaque strings, which is the right habit anyway (you never do arithmetic on an ID).
- **Cons:** Doesn't address enumeration or volume leakage. You have to remember to apply it to every ID property, including foreign keys like `customerId`.
- **Use when:** An existing app with `long` keys that may exceed 2^53, where changing the key type isn't worth it.

### Option 3: GUID primary keys, exposed directly

Use `Guid` keys and send them as is. JSON already carries a GUID as a string, so there is no precision problem. On SQL Server, EF Core generates sequential GUID values on the client, similar to `NEWSEQUENTIALID` [13]. The Npgsql PostgreSQL provider generates UUIDv7 values by default since version 9.0 [14]. In your own code, `Guid.CreateVersion7()` (available since .NET 9) creates a time-ordered UUID [15].

- **Grade:** 8/10. The default for new tables: no precision issue, not enumerable, and no mapping layer.
- **Pros:** Not guessable in practice (for random GUIDs), so enumeration and volume leaks go away. IDs can be created on the client or before insert, which helps with idempotent retries. One ID everywhere.
- **Cons:** 16 bytes instead of 4–8 in every index and foreign key. Long and awkward to read or say out loud. Random (v4) GUIDs insert in random index order; time-ordered GUIDs fix that but reveal creation time [9]. Changing an existing integer key to a GUID is a large migration.
- **Use when:** New tables, or any data where IDs reach the browser and counts or guessability matter.

### Option 4: Internal integer key plus a separate public ID

Keep a compact integer primary key for joins and foreign keys, and add a second column such as `PublicId` with a unique index. The API only ever accepts and returns the public ID. The public ID can be a random GUID, or a shorter random string, often with a type prefix like `ord_7Hq2…`, which makes IDs self-describing in logs and support tickets.

- **Grade:** 8/10. Best of both, at the cost of a second column and a lookup at every boundary.
- **Pros:** Internal joins stay fast and compact. External IDs are random and reveal nothing. You can change or rotate the public format without touching foreign keys. Internal IDs never leave the server, so they can't be enumerated.
- **Cons:** An extra unique index and an extra lookup per request (`WHERE PublicId = @id`). Two IDs per record to keep straight, and it's easy to leak the internal one by accident in a DTO.
- **Use when:** Existing integer-keyed schemas that now face the internet, or when you want short, prefixed, human-friendly IDs.

### Option 5: Obfuscate the integer (Sqids, Hashids, or encryption)

Encode the integer into a short string such as `"bM7h"` on the way out and decode it on the way in.

- **Grade:** 3/10. Looks random but isn't secure, and it gives you a false sense of protection.
- **Pros:** No schema change; short, URL-friendly IDs.
- **Cons:** Sqids itself says it "is not an encryption library" and lists sensitive data and user IDs as poor fits [16]. OWASP advises against encrypting identifiers as a protection [6]. Anyone who works out the alphabet or key gets enumeration back.
- **Use when:** Cosmetic short links where nothing depends on the ID being secret.

### Recommendation

| Option | Grade | Best for |
| --- | --- | --- |
| 3. GUID keys exposed directly | 8/10 | New tables in a new app |
| 4. Internal int key + public ID | 8/10 | Existing int-keyed schemas, or short readable IDs |
| 2. Integer sent as a JSON string | 6/10 | Quick fix for `long` IDs above 2^53 |
| 1. Integer as a JSON number | 5/10 | Internal tools with `int` keys |
| 5. Obfuscated integers | 3/10 | Cosmetic short links only |

For a new ASP.NET Core app, use **Option 3**, letting EF Core generate the GUIDs. For an existing app with integer keys, use **Option 4** for entities that reach the browser, and apply **Option 2** immediately to any `long` ID that could pass 2^53. In every case:
- treat IDs as opaque **strings** on the frontend;
- put the resource's ID in the route, filters in the query, and written data in the body;
- check ownership on every request that takes an ID, and return `404` when the caller may not see the object, so you don't confirm that it exists **[unverified]** (common practice; no primary source checked).

What you give up with Option 3: bigger indexes and IDs nobody can read aloud. With Option 4: an extra column, index and lookup, and the discipline to never expose the internal key.

## What the industry does

- **Object-level authorization on every access** is treated as non-negotiable. It's the first item in OWASP's API Security Top 10 [7], and OWASP's prevention advice includes writing tests for it [7].
- **Random IDs as a second layer.** OWASP recommends GUIDs or long random values for record IDs, as defense in depth [6][7].
- **IDs as strings in JSON.** Since JavaScript can't hold 64-bit integers in a `number` [1][3], APIs that use 64-bit IDs commonly send them as strings **[unverified]** (widely observed in public APIs; no primary source checked).
- **Time-ordered UUIDs** (UUIDv7) are replacing random ones for database keys, because they keep inserts in index order [9][14].
- **Prefixed public IDs.** Stripe prefixes object IDs with their type (for example `pi_` for a PaymentIntent) so a human can tell what an ID refers to and spot one passed in the wrong place [17].

**Anti-patterns:**
- Loading a record by ID from the request with no ownership or permission check.
- Relying on GUIDs, obfuscation, or encryption *instead of* authorization.
- Sending `long` IDs as JSON numbers when they can exceed 2^53 − 1.
- Doing arithmetic on IDs, or parsing them as numbers in the frontend.
- Using route constraints as validation, which returns `404` for bad input instead of `400` [11].
- Putting secret tokens in query strings, where they end up in logs and `Referer` headers [12].

## Concepts to look up

- **IEEE 754 double / `Number.MAX_SAFE_INTEGER`:** why JavaScript integers are exact only up to 2^53 − 1.
- **`BigInt`:** JavaScript's arbitrary-size integer type, which JSON doesn't support natively.
- **IDOR (Insecure Direct Object Reference) / BOLA (Broken Object Level Authorization):** accessing another user's object by changing an ID.
- **Resource-based authorization:** authorization that runs after the resource is loaded, so it can check ownership.
- **Trust boundary:** the line past which input can't be trusted; everything from the browser is outside it.
- **Enumeration attack:** iterating through predictable IDs to discover records.
- **Defense in depth:** stacking independent protections so one failure isn't a breach.
- **UUID versions (v4 random, v7 time-ordered):** the two GUID kinds you'll choose between, defined in RFC 9562.
- **Surrogate key vs natural key / public ID:** a database-generated key versus a meaningful one, and a separate externally visible identifier.
- **Security capability (bearer token):** an identifier whose possession alone grants access; record IDs must not be one.
- **Parameter binding:** how ASP.NET Core maps route, query, header and body values to handler parameters.

## Example

The code targets .NET 10 (LTS), ASP.NET Core minimal APIs and EF Core 10.

**Option 2: send a `long` ID as a string**, so the browser can't round it [4].

```csharp
public sealed record OrderDto(
    [property: JsonNumberHandling(JsonNumberHandling.WriteAsString | JsonNumberHandling.AllowReadingFromString)]
    long Id,                 // "9007199254740993" in JSON, exact in the browser
    decimal Total);

// Or for every number in every minimal API response (affects totals and counts too) [4][5]:
// builder.Services.ConfigureHttpJsonOptions(o =>
//     o.SerializerOptions.NumberHandling |= JsonNumberHandling.WriteAsString);
```

**Option 3 with an ownership check.** The ID comes from the route; the query filters by both ID and owner, so another user's order is simply "not found".

```csharp
public sealed class Order
{
    public Guid Id { get; set; }          // EF Core on SQL Server generates sequential GUIDs for this key [13]
    public string OwnerId { get; set; } = "";
    public decimal Total { get; set; }
}

app.MapGet("/orders/{id:guid}", async (Guid id, ClaimsPrincipal user, AppDbContext db, CancellationToken ct) =>
{
    var userId = user.FindFirstValue(ClaimTypes.NameIdentifier);

    var order = await db.Orders.AsNoTracking()
        .Where(o => o.Id == id && o.OwnerId == userId)   // object-level check, on every request [6][7]
        .Select(o => new { o.Id, o.Total })
        .FirstOrDefaultAsync(ct);

    return order is null ? Results.NotFound() : Results.Ok(order);
}).RequireAuthorization();
```

**When the rule is more than "owner only"**, load the object and ask `IAuthorizationService`, so the rule lives in one policy instead of in every query [8].

```csharp
app.MapDelete("/orders/{id:guid}", async (Guid id, ClaimsPrincipal user, AppDbContext db,
    IAuthorizationService authz, CancellationToken ct) =>
{
    var order = await db.Orders.FindAsync([id], ct);
    if (order is null) return Results.NotFound();

    var result = await authz.AuthorizeAsync(user, order, "CanDeleteOrder"); // policy with a handler for Order
    if (!result.Succeeded) return Results.NotFound();   // don't confirm the order exists

    db.Orders.Remove(order);
    await db.SaveChangesAsync(ct);
    return Results.NoContent();
}).RequireAuthorization();
```

**Option 4: internal key plus a public ID.** Only `PublicId` appears in routes and DTOs.

```csharp
public sealed class Customer
{
    public int Id { get; set; }                                  // internal: joins and foreign keys only
    public Guid PublicId { get; set; } = Guid.CreateVersion7();  // external: the only ID the API accepts [15]
    public string Name { get; set; } = "";
}

// In OnModelCreating: a unique index, since every request looks customers up by it.
modelBuilder.Entity<Customer>().HasIndex(c => c.PublicId).IsUnique();

app.MapGet("/customers/{publicId:guid}", async (Guid publicId, AppDbContext db, CancellationToken ct) =>
    await db.Customers.Where(c => c.PublicId == publicId)
        .Select(c => new { Id = c.PublicId, c.Name })            // never project the internal Id
        .FirstOrDefaultAsync(ct) is { } customer
            ? Results.Ok(customer)
            : Results.NotFound());
```

**On the frontend** (pseudocode), an ID is an opaque string:

```text
type OrderId = string                 // never number, never parsed, never incremented
order = await getJson("/orders/" + encodeURIComponent(id))
```

## Check your understanding

1. The backend sends `{"id": 9007199254740993}` and the frontend requests `/orders/9007199254740992`. Which component is wrong, and what are two fixes?

   <details><summary>Answer</summary>

   Nothing "crashes": the JSON text is correct, but the browser's `JSON.parse` turns it into a double, which can't represent integers above 2^53 − 1 exactly [1][3]. Fix it by sending the ID as a string (`JsonNumberHandling.WriteAsString` [4]), or by using an ID type that is a string in JSON anyway, such as a GUID.

   </details>

2. You switched to random GUIDs. Can you now drop the ownership check?

   <details><summary>Answer</summary>

   No. GUIDs make guessing hard, but IDs still leak through shared links, logs, screenshots and browser history. RFC 9562 says not to assume UUIDs are hard to guess and not to use them as security capabilities [9], and OWASP treats random IDs as defense in depth only [6].

   </details>

3. Why can't an `[Authorize(Policy = "OrderOwner")]` attribute on the endpoint enforce "only the owner can view this order"?

   <details><summary>Answer</summary>

   Attribute authorization runs before model binding and before the order is loaded, so the policy has no order to inspect [8]. Ownership needs resource-based authorization: load the order, then check it, either in the query or with `IAuthorizationService.AuthorizeAsync(user, order, policy)`.

   </details>

4. What does a UUIDv7 key reveal that a UUIDv4 key doesn't, and why might you choose it anyway?

   <details><summary>Answer</summary>

   It contains a 48-bit Unix timestamp, so anyone with the ID can see when the record was created [9]. You might still choose it because time-ordered keys insert in index order, which is kinder to B-tree indexes than random v4 values.

   </details>

5. `GET /orders/abc` hits a route defined as `/orders/{id:guid}` and returns 404. Is that the right status code?

   <details><summary>Answer</summary>

   Arguably not: the input is malformed, which is a `400`. Microsoft's routing docs say not to use constraints for input validation for exactly this reason [11]. Constraints exist to choose between routes. If you want `400`, bind a string, or let binding to `Guid` fail, and report a validation error.

   </details>

## Sources

1. MDN Web Docs (Mozilla), "Number.MAX_SAFE_INTEGER", § Description. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/MAX_SAFE_INTEGER
2. MDN Web Docs (Mozilla), "BigInt", § "Use within JSON". https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt
3. T. Bray (ed.), "The JavaScript Object Notation (JSON) Data Interchange Format", RFC 8259, IETF, 2017, §6 "Numbers". https://www.rfc-editor.org/rfc/rfc8259#section-6
4. Microsoft, "How to allow some kinds of invalid JSON with System.Text.Json", § "Allow or write numbers in quotes". https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/invalid-json
5. Microsoft, "Create responses in Minimal API applications", § "Configure JSON serialization options globally". https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/responses
6. OWASP, "Insecure Direct Object Reference Prevention Cheat Sheet", § "Mitigation", § "Identifier complexity". https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html
7. OWASP, "API1:2023 Broken Object Level Authorization", OWASP API Security Top 10 2023. https://api-security.owasp.org/editions/2023/en/0xa1-broken-object-level-authorization
8. Microsoft, "Resource-based authorization in ASP.NET Core", § "Use imperative authorization". https://learn.microsoft.com/en-us/aspnet/core/security/authorization/resource-based
9. K. Davis, B. Peabody, P. Leach, "Universally Unique IDentifiers (UUIDs)", RFC 9562, IETF, 2024: §5.7 (UUIDv7), §8 (Security Considerations). https://www.rfc-editor.org/rfc/rfc9562
10. Microsoft, "Parameter binding in Minimal API applications", § "Binding precedence". https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/parameter-binding
11. Microsoft, "Routing in ASP.NET Core", § "Route constraints". https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing
12. OWASP, "Information exposure through query strings in URL". https://owasp.org/www-community/vulnerabilities/Information_exposure_through_query_strings_in_url
13. Microsoft, "Microsoft SQL Server Database Provider: Value Generation", EF Core docs, § "GUIDs". https://learn.microsoft.com/en-us/ef/core/providers/sql-server/value-generation
14. Npgsql, "Npgsql EF Core 9.0 release notes", § "UUIDv7 GUIDs are generated by default". https://www.npgsql.org/efcore/release-notes/9.0.html
15. Microsoft, "Guid.CreateVersion7 Method", .NET API reference. https://learn.microsoft.com/en-us/dotnet/api/system.guid.createversion7
16. Sqids, project home page, § "Not Good For". https://sqids.org/
17. Paul Asjes (Stripe developer advocate), "Designing APIs for humans: Object IDs", dev.to/stripe, 2022. Community source. https://dev.to/stripe/designing-apis-for-humans-object-ids-3o5a

**Unverified claims in this entry:**
- Returning `404` instead of `403` to avoid confirming that an object exists is common practice. No primary source was checked.
- APIs with 64-bit IDs commonly send them as strings. Widely observed; no primary source was checked.

## Related

- [Passing IDs between services](../distributed-systems/passing-ids-between-services.md): the same question across a service boundary, plus correlation IDs.
