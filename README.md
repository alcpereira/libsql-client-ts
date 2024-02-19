<p align="center">
  <a href="https://docs.turso.tech/sdk/ts/quickstart">
    <img alt="Turso + TypeScript cover" src="https://github.com/tursodatabase/libsql-client-ts/assets/950181/293186fa-ffe4-4dfb-84fa-3f668b991253" width="1000">
    <h3 align="center">Turso + TypeScript / JS</h3>
  </a>
</p>

<p align="center">
  Turso is a SQLite-compatible database built on libSQL.
</p>

<p align="center">
  <a href="https://turso.tech"><strong>Turso</strong></a> ·
  <a href="https://docs.turso.tech/quickstart"><strong>Quickstart</strong></a> ·
  <a href="/examples"><strong>Examples</strong></a> ·
  <a href="https://docs.turso.tech"><strong>Docs</strong></a> ·
  <a href="https://discord.com/invite/4B5D7hYwub"><strong>Discord</strong></a> ·
  <a href="https://blog.turso.tech/"><strong>Tutorials</strong></a>
</p>

<p align="center">
  <a href="https://www.npmjs.com/@libsql/client">
    <img src="https://badge.fury.io/js/@libsql%2Fclient.svg" alt="npm version" title="npm version" />
  </a>
</p>

---

## Install

```bash
npm install @libsql/client
```

## Import

This library supports multiple runtimes, including Node.js, Cloudflare Workers, Deno, and experimental WebAssembly.

**Make sure you import the correct client for your environment:**

|                                  | Example                                                                      |
| -------------------------------- | ---------------------------------------------------------------------------- |
| Node                             | `import { createClient } from "@libsql/client"`                              |
| Browsers                         | `import { createClient } from "@libsql/client/web"`                          |
| Edge <br />(Cloudflare/Vercel)   | `import { createClient } from "@libsql/client/web"`                          |
| Deno                             | `import { createClient } from "https://esm.sh/@libsql/client@[version]/web"` |
| WebAssembly <br />(Experimental) | `import { createClient } from "@libsql/client-wasm"`                         |

## Connect

You can use this library to connect to [Turso](#turso), [local SQLite](#local-sqlite), [Embedded Replicas](#embedded-replicas), and [libSQL server](#libsql-server).

### Turso

Follow the [Turso Quickstart](https://docs.turso.tech/quickstart) to create an account, database, auth token, and connect to the shell to create a database schema.

```ts
import { createClient } from "@libsql/client";

const client = createClient({
  url: "libsql://...",
  authToken: "...",
});
```

### Local SQLite

You can use a new or existing local SQLite file on your machine:

```ts
import { createClient } from "@libsql/client";

const client = createClient({
  url: "file:dev.db",
});
```

### Embedded Replicas

You can achieve zero-latency network requests by using [Embedded Replicas](https://docs.turso.tech/features/embedded-replicas) that are local-first, and sync with a remote database (Turso or [libSQL Server](#libsql-server)):

```ts
import { createClient } from "@libsql/client";

const client = createClient({
  url: "file:replica.db",
  syncUrl: "libsql://...",
  authToken: "...",
});
```

Embedded Replicas require access to the filesystem, **serverless environments aren't compatible**.

### libSQL Server

You can use this package with [libSQL server](https://github.com/tursodatabase/libsql/tree/main/libsql-server) directly using one of the methods [here](https://github.com/tursodatabase/libsql/blob/main/docs/BUILD-RUN.md):

```ts
import { createClient } from "@libsql/client";

const client = createClient({
  url: "http://127.0.0.1:8080",
});
```

## Execute

### Simple query

You can execute a SQL query against your existing database by calling `execute()`:

```ts
const result = await client.execute("SELECT * FROM users");
```

### Arguments

If you need to use placeholders for values, you can do that using positional and named `args`:

```ts
const result = await client.execute({
  sql: "SELECT * FROM users WHERE id = ?",
  args: [1],
});

const result = await client.execute({
  sql: "INSERT INTO users VALUES (:name)",
  args: { name: "Iku" },
});
```

_You can use `:` (shown above), `@` and `$` to declare named arguments._

### Sync

If you're using [Embedded Replicas](#embedded-replicas), you should call `sync()` in the background whenever your application wants to sync local embedded replica with the remote database. For example, you can call it every 5 minutes or every time the application starts.

```ts
import { createClient } from "@libsql/client";

const client = createClient({
  url: "file:replica.db",
  syncUrl: "libsql://...",
  authToken: "...",
});

await client.sync();
```

## Transaction Modes

| Mode       | SQLite command               | Description                                                                                                                                                                                             |
| ---------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `write`    | `BEGIN IMMEDIATE`            | The transaction may execute statements that read and write data.<br />Write transactions executed on a replica are forwarded to the primary instance, and can't operate in parallel.                    |
| `read`     | `BEGIN TRANSACTION READONLY` | The transaction may only execute statements that read data (select).<br />Read transactions can occur on replicas, and can operate in parallel with other read transactions.                            |
| `deferred` | `BEGIN DEFERRED`             | The transaction starts in read mode, then changes to write as soon as a write statement is executed.<br />This mode change may fail if there is a write transaction currently executing on the primary. |

### Batch

A batch runs multiple SQL statements in a row, managed by the backend in an automatic transaction. Success commits all changes; failure triggers a complete rollback without alterations:

```ts
const result = await client.batch(
  [
    {
      sql: "INSERT INTO users VALUES (?)",
      args: ["Iku"],
    },
    {
      sql: "INSERT INTO users VALUES (?)",
      args: ["Iku 2"],
    },
  ],
  "write"
);
```

### Interactive Transactions

SQLite's interactive transactions maintain data consistency for read-write operations, allowing controlled commits or rollbacks and isolating changes from other client activities.

| Method       | Description                                                         |
| ------------ | ------------------------------------------------------------------- |
| `execute()`  | Similar to `execute()` except within the context of the transaction |
| `commit()`   | Commits all write statements in the transaction                     |
| `rollback()` | Rolls back the entire transaction                                   |
| `close()`    | Immediately stops the transaction                                   |

```ts
try {
  const userId = "user123";
  const withdrawalAmount = 500;

  const transaction = await client.transaction("write");

  const balanceResult = await transaction.execute({
    sql: "SELECT balance FROM accounts WHERE userId = ?",
    args: [userId],
  });

  const currentBalance = balanceResult.rows[0]["balance"] as number;

  if (currentBalance >= withdrawalAmount) {
    await transaction.execute({
      sql: "UPDATE accounts SET balance = balance - ? WHERE userId = ?",
      args: [withdrawalAmount, userId],
    });
  } else {
    console.log("Insufficient funds");
    await transaction.rollback();
    return;
  }

  await transaction.commit();
} catch (e) {
  console.error(e);
}
```

## Quickstart Guides

- [Next.js](https://docs.turso.tech/sdk/ts/guides/nextjs)
- [Remix](https://docs.turso.tech/sdk/ts/guides/remix)
- [Astro](https://docs.turso.tech/sdk/ts/guides/astro)
- [Nuxt](https://docs.turso.tech/sdk/ts/guides/nuxt)
- [Qwik](https://docs.turso.tech/sdk/ts/guides/qwik)
- [SvelteKit](https://docs.turso.tech/sdk/ts/guides/sveltekit)
- [Quasar](https://docs.turso.tech/sdk/ts/guides/quasar)

## License

This project is licensed under the MIT license.
