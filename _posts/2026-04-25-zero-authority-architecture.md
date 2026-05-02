---
layout: post
title: The Zero Authority Architecture
---
# TL;DR

Traditional internet-facing apps give the middleware standing authority to read and write the database. If that layer is compromised, everything is exposed.

The Zero Authority Architecture pattern removes this standing authority entirely. It is also the natural way to implement a Postgres-backed PaaS.

* The user holds an opaque, revocable credential (e.g. cookie/passkey-backed token).
* Each request carries a short-lived, scoped capability describing exactly what operation is allowed.
* The middleware is a dumb broker: it cannot read or mint authority, only forward requests.
* The database validates the request, establishes transaction-local identity (tenant/user), and executes the operation.
* Authorization and business logic reside in the database (e.g. via RLS, policies, triggers).
* Operations are invoked by inserting into command tables (schema-as-API), providing built-in Event tracing.

Result:

Even if the application server is fully compromised, it cannot dump the database — it can only act within the scope of live, user-authorised requests. Security becomes a property of the architecture, not the discipline of the programmer.

This can be implemented in many systems; Postgres provides an especially clean realisation using session variables, RLS, and trigger-driven command tables.

State-changing operations are invoked by inserting a row into a table whose trigger performs the operation.

This is a flexible design pattern. As a demonstration, the exposition also yields an almost trivial multitenant architecture.

# Introduction

The _Zero Authority Architecture_ employs a minimal middleware layer to exchange signed (optionally, encrypted) authorised operation requests between client and back end. Back end operations are all executed with minimal necessary authority.

The application server does not have the authority to query data; it only has the ability to present requests that may or may not be authorised by the database. A compromised application server should not be able to read your database at rest.

This post describes how to implement such a pattern in Postgres. The pattern offers many advantages:

- **No standing authority**  
  The application server cannot access data without a user-authorised request.
- **Reduced blast radius**  
  A compromised middleware cannot dump the database; it can only act within the scope of live, user-authorised requests.
- **Strict least privilege**  
  Every operation runs with minimal, request-scoped authority.
- **Simple middleware**  
  Authentication + forwarding; minimal logic to get wrong.
- **Centralised logic and authorisation**  
  Policies and behaviour live in the database.
- **Uniform API**  
  Operations can be made available to any client, depending on authorisation.
- **Built-in auditability**
  Commands are structured and optionally persistable.
- **Context-driven isolation**
  All operations run with an explicit request context (e.g. tenant/user), so concerns like multitenancy become automatic consequences of policy rather than logic scattered through the application.

Note: This is a design pattern, not a prescription; different deployments may move parts of the validation or execution outside the database for operational reasons.

After describing the pattern, I will discuss how the pattern can be applied in various ways; how the design is language and client platform neutral, and note how it can be migrated to from a traditional (say, web) app gradually.

# The Minimal Authority Principle

It is widely accepted (e.g. [NIST SP 800-53](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-53r5.pdf)) that secure systems should ensure every operation is executed with the minimal authority required. The benefit is simple: any compromise is limited to the scope of that authority.

The Zero Authority Architecture pushes this further. Rather than granting components minimal standing authority, it eliminates standing authority entirely. Authority is derived only at the point of request, and only for the duration of that request.

Such a design falls out naturally from the ZAA architecture because the back end starts processing each request with minimal authority. A small, secure broker provides a single way to raise authority just by as much as is needed, only at the point of need.

# The Zero Authority Architecture in Postgres

The client's initial login is handled in the back end, with the middleware only forwarding public key encrypted exchanges between the two. Following login, the client is provided with a normal session token.

![A Zero Authority Architecture request](/assets/img/request_scoped_authority.svg)

Subsequent requests provide the token, a timestamp preventing hostile replay attacks, an identifier for the requested operation, and the arguments to that operation. The token must be revocable and time-bounded, and requests must be protected against replay (e.g. via timestamps and/or nonces).

Postgres decrypts the request, validates the token and timestamp, sets the user (and in this example, the tenant_id) as a session variable, and tries to invoke the operation. e.g.

```SQL
-- inside broker function
BEGIN;
SET LOCAL app.user_id = $user_id;
SET LOCAL app.tenant_id = $tenant_id;
-- RLS will validate this
INSERT INTO function_schema.$function_name(…) VALUES(…);
COMMIT;
```

Observe that the function_name must be the name of a function in the function_schema. This is a simple way to ensure that only functions that are intended to be invoked by the middleware can be invoked.

Note that state-changing operations write the request to a table; the actual function is a trigger on that table. In this way, event sourcing and logging are incorporated into the request execution. This aspect of the pattern is optional, but recommended, since it naturally expresses security concerns and removes the need for separate event sourcing and logging of requests.

Non state-changing operations ("queries") will often be directly executed, rather than via a trigger. These operations might be function invocations, or might be queries either in SQL or through something like GraphQL. If it is desired to log queries, they can be executed in the same way as state-changing operations.

The trigger on the table will be executed contingent on standard Postgres Row Level Security authorisation logic, e.g.:

```SQL
CREATE POLICY 
  can_create_invoice 
ON 
  function_tables.create_invoice
FOR INSERT
WITH CHECK (
  role_supported(
    current_setting('app.user_id', true),
    'create invoice'
  )
  AND current_setting('app.tenant_id', true) = tenant_id
);
```

In this example, there is something like a many:many relation between users and capabilities which the role_supported function draws on for authorisation. But any sort of authorisation regime can be used.

Side effects, in the form of table updates, can also be gated by RLS policies in a similar manner.

Side effects outside of Postgres state updates (e.g. sending email) can be handled either by a trigger sending a message using Postgres's LISTEN/NOTIFY feature, or by some other similar means ([Supabase's REALTIME project](https://github.com/supabase/realtime) may be a better alternative for many purposes than LISTEN/NOTIFY.) Note that since any such side effect starts with a table write, all operations can still be gated using the Postgres authorisation architecture.

# Synchronous vs Asynchronous requests

All operations over a network are inherently asynchronous. Various abstractions (TCP/HTTP/database connections/…) conceal this reality, but are inevitably leaky abstractions.

ZAA works naturally with asynchronous execution. The client submits a request and provides a channel for the response (e.g. WebSocket or HTTP connection). The middleware tags the request and routes the eventual response back to the caller.

This model also allows some operations to be executed asynchronously by worker pools, rather than immediately within the request.

# Discussion

## Access Modes

Note how simple it is to implement a multitenant architecture; all that is required is a RLS check on the tenant_id column of any multitenant table.

Note also that this is a multitenant architecture that is trivial for the client to *use*: it can treat every request as belonging to a single-tenant database; the database does all the work.

Authorisation is also an access mode. Other sorts of access modes are possible: if you want to only allow some values to be seen during business hours, say, this can also be enforced using RLS.

## Client neutrality

The original intent of the SQL database was that it should form a neutral data store, accessible from many different clients for different purposes. Centering the database in the internet-facing application recovers this feature.

Client can be written in any language, on any platform. Different clients can easily be given very different access rights and other modes. That the internet-facing middleware provides minimal authority access does not prevent an internal application, accessing the database using a different role, from having whatever privileges are required.

## Implementation language neutrality

Those unfamiliar with Postgres may not realise how friendly it is to scripting in many languages. At time of writing (mid 2026), the languages that can be used to write triggers, stored procedures, custom functions and many other things (custom types!) include at least:

Built-in languages:
- PL/PGSQL
- TCL
- Perl
- Python
- C

- And many [installable languages](https://wiki.postgresql.org/wiki/PL_Matrix):
- Java/Clojure
- PHP
- R (run statistical analyses directly in the database!)
- sh
- Javascript
- Julia
- C#/F#/.Net
- Haskell
- Lua
- Rust
- Scheme

## Traditional web clients

Traditional web clients, where logic resides mostly in the middleware layer, can also use this pattern. The middleware gateway to the database can still be a separate service, or it can be part of the application logic middleware. In the latter case, most of the security advantages of this pattern remain if the middleware only keeps access tokens in the browser as a cookie.

## Testing and replay

The recommended pattern incorporates Event Sourcing as a matter of course, facilitating the traditional Event Sourcing advantages, such as replay and testing.

## Gradual migration

Existing logic-heavy-middleware applications can migrate to this pattern gradually. It is not necessary to have business logic in the database before the RLS-based security can be implemented.

## Other implementations

Postgres is particularly well-suited to this pattern. Other highly programmable relational stores will also work well. The back end here could be replaced with one based on SQLite, and Oracle and SQL Server can be programmed in Python.

Another possibility pointed out to me by my friend [Tonio](https://www.linkedin.com/in/tonioloewald/) is that [this pattern works great in any strongly sandboxed language](https://www.linkedin.com/feed/update/urn:li:activity:7455127984388124672/?dashCommentUrn=urn%3Ali%3Afsd_comment%3A%287455287608944951296%2Curn%3Ali%3Aactivity%3A7455127984388124672%29) — Javascript and WASM being the most obvious examples.

## Great for small business software

Many engineers will instinctively recoil from this pattern as "inefficient" because it moves business logic into the database.

For typical small business software, however, minimising storage, bandwidth, and CPU time is rarely the primary concern. It is often acceptable to be profligate with resources if it produces software that is cheaper to build and more flexible.

I'll also note in passing here that for small business software, you can lean into Postgres features that are a little "inefficient", particularly [composite types](https://www.postgresql.org/docs/current/rowtypes.html) note the ["inefficiency"](https://lydb.xyz/postgres-types/).

## Also great for large business software.

Postgres is open source that is free to use and cheap to deploy.

Many engineers thinking about larger business software will recoil from this pattern because it is "inefficient" to run business logic in the database. Yet is it?

Large software products already routinely employ multiple middleware and database servers. If this pattern leads to fewer middleware servers and more Postgres servers, is that necessarily a problem?

Moreover, the simple in-depth security by default offered by this architecture is a significant advantage in large-scale systems, as is the language and plaform neutrality. Your mobile developers can write their business logic in JavaScript or Kotlin, or they can call the business logic written by the back end engineers in Python.

This piece hasn't even discussed the deep architectural possibilities that Postgres offers. It is straightforward, for example, to set up remote access between Postgres servers, so that they can transparently join across data stores or invoke remote functions.

Postgres is an ideal neutral articulation point between heterogeneous data stores and services.

## Related code

Folks on the Postgres list seem to see this as unremarkable. Merlin Moncure said he's been developing in this style for a while, and he has some [interesting code to use with this pattern](https://www.postgresql.org/message-id/CAHyXU0yNBHF%3DyJG8-XAasedUagG%3D%2BPODje_RXZVuefWqhfuvdQ%40mail.gmail.com).

# Conclusion

The Zero Authority Architecture is a small shift in where we place trust, but it has large consequences. This design deliberately trades some conventional layering for stronger security and simpler reasoning about authority.

By removing standing authority from the application layer, we stop relying on the middleware to “do the right thing.” Instead, every request must prove its authority, and that authority is established and enforced at the point where the data actually lives.

Postgres turns out to be a particularly good substrate for this pattern. Session-local context, Row Level Security, and trigger-driven execution provide a coherent model in which identity, authorisation, and behaviour all reside in one place. The result is a system where security, multitenancy, and business logic are expressed declaratively, rather than being scattered across services.

This is not a silver bullet. It introduces its own complexities, and not every system will benefit from pushing logic this far into the database. But it does demonstrate that a different balance is possible: one in which the database is not merely a store, but the authority boundary of the system.

This post is a first draft. Feedback is very welcome—particularly from those with experience pushing Postgres beyond its traditional role.