---
title: "Catching silent column drops in PostgreSQL views with a Prisma test"
description: "How a 20-line E2E test prevents a subtle bug where migrations recreate views without all expected columns, and Prisma returns null instead of throwing."
pubDate: 2026-04-17
tags: ["postgresql", "prisma", "testing", "nestjs"]
---

A colleague added a column to a PostgreSQL view. Two days later, a migration on a different branch recreated the same view without it. The column returned `null` in production and we caught it by accident, looking at a value that didn't add up.

The bug was silent for several reasons. Prisma doesn't validate columns at runtime: if a column exists in the schema but not in the database, it just returns `null`. TypeScript compiles fine because types come from the schema file, not from the actual database. And nobody manually checks all 70 columns of a denormalized view after every merge.

## Why this happens

In PostgreSQL you can't `ALTER VIEW` to add a column. You have to drop and recreate the view with `CREATE OR REPLACE VIEW` that includes all columns, old and new. Every migration that modifies a view contains its entire definition, rewritten from scratch.

In a team, this creates a subtle problem. Two developers working on separate branches both need to add a column to the same view. Each creates their own migration with a full `CREATE OR REPLACE VIEW`. When the branches get merged, the second migration overwrites the first. The column added by the first developer disappears.

## The fix: comparing database and schema

Before writing the test, I considered diffing the Prisma schema against the database with `prisma db pull`, but the output format made comparison awkward. Querying `information_schema.columns` directly turned out to be simpler.

The idea: PostgreSQL exposes `information_schema.columns`, a system view that lists all columns of all tables and views. Query it for the actual columns, compare with the columns Prisma expects, fail if they don't match.

```typescript
import { type INestApplication } from '@nestjs/common';
import { Prisma } from '@prisma/client';

import { type PrismaService } from '@/prisma/prisma.service';
import { setupTestApp, teardownTestApp } from '@/test/e2e/helpers';

describe('ProductSummaryView columns integrity', () => {
    let app: INestApplication;
    let prisma: PrismaService;

    beforeAll(async () => ({ app, prisma } = await setupTestApp()));
    afterAll(async () => await teardownTestApp(app));

    it('database view columns should match Prisma schema', async () => {
        const dbColumns = await prisma.$queryRaw<
            Array<{ column_name: string }>
        >`
            SELECT column_name
            FROM information_schema.columns
            WHERE table_schema = 'public'
              AND table_name = 'ProductSummaryView'
            ORDER BY column_name
        `;

        const schemaColumns = Object.keys(
            Prisma.ProductSummaryViewScalarFieldEnum,
        ).sort();

        expect(dbColumns.map((c) => c.column_name)).toStrictEqual(
            schemaColumns,
        );
    });
});
```

A few details on the implementation:

- **`ScalarFieldEnum` excludes relations.** Prisma generates `ScalarFieldEnum` only for scalar fields, excluding relations. This is exactly what we need, because `information_schema.columns` only lists physical columns, not Prisma's logical relations.
- **The sort matters.** `information_schema.columns` returns columns in definition order (or alphabetically with `ORDER BY`). `Object.keys()` on a JavaScript object follows insertion order. Sorting both lists alphabetically makes the comparison deterministic.
- **No test data required.** The query on `information_schema.columns` works on an empty view because it reads schema metadata, not row data.

## What happens when it fails

The error message is clear because it uses `toStrictEqual` on two string arrays. If a column is missing, the output shows exactly which one:

```
Expected: ["createdAt", "description", "name", "status", ...]
Received: ["createdAt", "description", "status", ...]
```

The developer sees immediately which column disappeared and can go fix the migration.

## Maintenance cost

The test is self-contained: no fixtures, no mocks, no special setup beyond the NestJS app bootstrap that's already there for other E2E tests. About 20 lines of actual code. It never changes as long as the view keeps its name. If the view gets renamed or dropped, the test stops compiling (because the `ScalarFieldEnum` no longer exists), which is a good reminder to update or remove it. The query itself takes a few milliseconds. I have one test per view in the project; copying them is a matter of changing the view name and the `ScalarFieldEnum`.

Note: this approach works for regular views. If your project uses materialized views, `information_schema.columns` won't include them. You'd need to query `pg_attribute` joined with `pg_class` instead.

If you use Prisma with PostgreSQL views and you've lost columns after a migration, try adding a test like this. It takes five minutes and can save you hours of debugging.
