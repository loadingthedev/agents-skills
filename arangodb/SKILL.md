---
name: arangodb
description: Use this skill when the user asks to build ArangoDB-based applications in any project that uses ArangoDB and `arangojs`.
license: Complete terms in LICENSE.txt
---


## Purpose

Use this skill when building, reviewing, debugging, optimizing, or explaining ArangoDB-based applications in any project that uses ArangoDB and `arangojs`.

This skill is for development-time work, including:

- data modeling
- collection design
- document and edge patterns
- AQL querying
- CRUD operations
- indexes and search
- graph traversals and path queries
- schema validation and computed values
- transactions and consistency
- performance tuning and profiling
- `arangojs` usage from Node.js / TypeScript
- common patterns, anti-patterns, and troubleshooting

This skill is written to be practical for everyday backend development.

---

## Scope

This skill covers the parts of ArangoDB that are most relevant during application development:

1. **Documents and collections**
2. **AQL**
3. **Indexes**
4. **ArangoSearch / inverted indexing**
5. **Graphs**
6. **Transactions**
7. **Schema validation**
8. **Computed values**
9. **Performance diagnostics**
10. **`arangojs` driver patterns**

---

## Operating principles for the agent

### 1) Start with the user’s data shape

Before writing queries, identify:

- document collections
- edge collections
- scoping rules (optional)
- soft-delete fields
- lifecycle timestamps
- unique constraints
- expected read patterns
- expected write patterns

Do not jump straight to query syntax before understanding the data model.

### 2) Prefer simple AQL first

Build the simplest correct query first:

- base `FOR`
- required `FILTER`
- minimal `RETURN`
- then add `SORT`
- then `LIMIT`
- then subqueries or aggregations
- then optimization

### 3) Bind values, compose syntax

**Bind parameters are for values, not for AQL syntax.**

In `arangojs`:

- values such as `id`, `q`, `offset`, `limit`, `status`, and `category` should be bind parameters
- query fragments such as `FILTER`, `SORT`, or `LIMIT` must be composed using nested `aql` fragments

### 4) Whitelist dynamic fields

If sorting or filtering by field name is dynamic:

- whitelist known fields
- map to fixed AQL fragments
- never inject raw user-controlled field names into query text

### 5) Return minimal shapes

Return only what the caller needs.
Avoid returning full documents unless explicitly necessary.

### 6) Optimize from workload, not theory

Choose indexes, schema, and query shape according to:

- frequent filters
- joins
- graph traversal depth
- update rates
- data size
- latency requirements

---

# 1. ArangoDB mental model

ArangoDB is a multi-model database centered on JSON documents.
The same engine supports:

- **document model**
- **graph model**
- **search model**
- **key/value-style lookups**

Core primitives:

- **document collections** store regular JSON-like documents
- **edge collections** store relationships using `_from` and `_to`
- **AQL** is the main query language for reads and writes
- **indexes** accelerate specific access patterns
- **Views / inverted indexing / ArangoSearch** support advanced search
- **graphs** support traversals and path queries

---

# 2. Development workflow

Use this order during implementation:

1. Define the document shape.
2. Decide whether relationships should be embedded or modeled as edges.
3. Add required collection-level features:
   - schema validation
   - computed values
   - key strategy
4. Write the minimal AQL query.
5. Add indexes only for proven access paths.
6. Run `EXPLAIN` / profiling when performance matters.
7. Revisit the model if queries become unnatural or index-heavy.

---

# 3. Data modeling guidance

## 3.1 Documents

Each document has built-in attributes such as:

- `_key`
- `_id`
- `_rev`

For edge documents:

- `_from`
- `_to`

Use your own top-level fields for:

- application scoping (optional)
- lifecycle state
- soft delete
- timestamps
- ownership
- domain-specific attributes

Example document:

```json
{
  "_key": "brand-001",
  "name": "Acme",
  "discount": 10,
  "ownerId": "users/42",
  "status": "active",
  "isDeleted": false,
  "createdAt": "2026-03-31T12:00:00Z",
  "updatedAt": "2026-03-31T12:00:00Z"
}
```

## 3.2 Embed vs reference vs edge

### Embed when:

- the child data is always loaded with the parent
- the child has no independent lifecycle
- duplication is acceptable

### Reference by id when:

- relation is simple
- traversal semantics are not needed
- joins are occasional and shallow

### Use an edge collection when:

- the relation is first-class
- you need multi-hop traversal
- the relation has attributes
- shortest path / path enumeration matters
- relation-centric queries are common

## 3.3 Logical scoping (optional)

Some applications need a logical boundary such as:

- workspace
- organization
- account
- project
- environment

If the application has that boundary:

- store the scoping field explicitly
- apply the scoping filter consistently
- include the scoping field in uniqueness rules when needed

Generic pattern:

```aql
FILTER doc.scopeId == @scopeId
```

## 3.4 Soft delete

Common pattern:

```aql
FILTER doc.isDeleted == false
```

If most production queries exclude deleted records:

- consistently apply the filter
- consider indexes that include commonly filtered fields

## 3.5 Key strategy

Prefer `_key` values that are:

- deterministic when useful
- stable
- not overly random for large sequential inserts

Avoid relying on accidental formatting differences.

---

# 4. Collection design patterns

## 4.1 Regular document collection

Use for entities like:

- users
- brands
- products
- orders
- audit events

## 4.2 Edge collection

Use for:

- ownership
- follows / friends / memberships
- product-category links
- dependencies
- graph relationships with metadata

Example edge:

```json
{
  "_from": "users/42",
  "_to": "brands/brand-001",
  "role": "owner",
  "createdAt": "2026-03-31T12:00:00Z"
}
```

## 4.3 Time-based data

For event-like data:

- include a timestamp
- design queries around recent windows
- consider TTL index only when automatic expiry is desired

---

# 5. AQL fundamentals

## 5.1 Standard read shape

```aql
FOR doc IN collection
  FILTER ...
  SORT ...
  LIMIT ...
  RETURN ...
```

## 5.2 Standard write shapes

```aql
INSERT ... INTO collection
RETURN NEW
```

```aql
UPDATE ... IN collection
RETURN NEW
```

```aql
REPLACE ... IN collection
RETURN NEW
```

```aql
REMOVE ... IN collection
RETURN OLD
```

```aql
UPSERT ... INSERT ... UPDATE ... IN collection
RETURN NEW
```

## 5.3 Read examples

### Get active brands

```aql
FOR brand IN product_brands
  FILTER brand.isDeleted == false
    RETURN {
    id: brand._key,
    name: brand.name,
    discount: brand.discount
  }
```

### Search brands by name

```aql
FOR brand IN product_brands
  FILTER brand.isDeleted == false
    FILTER LIKE(LOWER(brand.name), CONCAT('%', LOWER(@q), '%'))
  SORT brand.createdAt DESC
  RETURN {
    id: brand._key,
    name: brand.name,
    discount: brand.discount
  }
```

### Paginated list with total count

```aql
LET total = FIRST(
  FOR brand IN product_brands
    FILTER brand.isDeleted == false
        FILTER @q == null OR LIKE(LOWER(brand.name), CONCAT('%', LOWER(@q), '%'))
    COLLECT WITH COUNT INTO count
    RETURN count
)

LET rows = (
  FOR brand IN product_brands
    FILTER brand.isDeleted == false
        FILTER @q == null OR LIKE(LOWER(brand.name), CONCAT('%', LOWER(@q), '%'))
    SORT brand.createdAt DESC
    LIMIT @offset, @limit
    RETURN {
      id: brand._key,
      name: brand.name,
      discount: brand.discount
    }
)

RETURN { total, rows }
```

## 5.4 Write examples

### Insert

```aql
INSERT {
  name: @name,
  discount: @discount,
  isDeleted: false,
  createdAt: DATE_ISO8601(DATE_NOW()),
  updatedAt: DATE_ISO8601(DATE_NOW())
} INTO product_brands
RETURN NEW
```

### Update specific fields

```aql
FOR brand IN product_brands
  FILTER brand._key == @id
    FILTER brand.isDeleted == false
  UPDATE brand WITH {
    name: @name,
    discount: @discount,
    updatedAt: DATE_ISO8601(DATE_NOW())
  } IN product_brands
  RETURN NEW
```

### Soft delete

```aql
FOR brand IN product_brands
  FILTER brand._key == @id
    FILTER brand.isDeleted == false
  UPDATE brand WITH {
    isDeleted: true,
    updatedAt: DATE_ISO8601(DATE_NOW())
  } IN product_brands
  RETURN NEW
```

### UPSERT

```aql
UPSERT { name: @name }
  INSERT {
      name: @name,
    discount: @discount,
    isDeleted: false,
    createdAt: DATE_ISO8601(DATE_NOW()),
    updatedAt: DATE_ISO8601(DATE_NOW())
  }
  UPDATE {
    discount: @discount,
    isDeleted: false,
    updatedAt: DATE_ISO8601(DATE_NOW())
  }
IN product_brands
RETURN NEW
```

---

# 6. AQL patterns you will use often

## 6.1 Single-document lookup by key

```aql
FOR doc IN product_brands
  FILTER doc._key == @id
  RETURN doc
```

## 6.2 Join-like lookup with subquery

```aql
FOR brand IN product_brands
    LET owner = FIRST(
    FOR user IN users
      FILTER user._key == PARSE_IDENTIFIER(brand.ownerId).key
      RETURN {
        id: user._key,
        firstName: user.firstName,
        lastName: user.lastName,
        email: user.email
      }
  )
  RETURN {
    id: brand._key,
    name: brand.name,
    owner
  }
```

## 6.3 Distinct values / grouping

```aql
FOR brand IN product_brands
    COLLECT discount = brand.discount WITH COUNT INTO count
  SORT discount ASC
  RETURN { discount, count }
```

## 6.4 Existence check

```aql
RETURN LENGTH(
  FOR brand IN product_brands
        FILTER LOWER(brand.name) == LOWER(@name)
    FILTER brand.isDeleted == false
    LIMIT 1
    RETURN 1
) > 0
```

## 6.5 Batch lookup by keys

```aql
FOR key IN @keys
  LET doc = DOCUMENT(product_brands, key)
  FILTER doc != null
  RETURN doc
```

## 6.6 Bulk insert

```aql
FOR doc IN @docs
  INSERT doc INTO product_brands
  RETURN NEW._key
```

## 6.7 Bulk update

```aql
FOR patch IN @patches
  LET doc = DOCUMENT(product_brands, patch._key)
  FILTER doc != null
  UPDATE doc WITH patch IN product_brands
  RETURN NEW
```

---

# 7. AQL dos and don’ts

## Do

- use bind parameters for values
- use `COLLECT WITH COUNT INTO` for counts
- keep `RETURN` shapes minimal
- push selective `FILTER` conditions as early as possible
- reuse filter logic in count and data subqueries
- use `PRUNE` in traversals when you want to stop exploring a path early
- profile important queries

## Do not

- inline user input into AQL strings
- dynamically inject raw `FILTER`, `SORT`, or `LIMIT` strings into `aql`
- use `FILTER` when you really need `PRUNE` in traversals
- return large documents when only 3 fields are needed
- add indexes “just in case”
- assume full collection scans are acceptable in hot paths

---

# 8. Index strategy

Indexes speed up reads but cost:

- disk
- RAM
- write throughput
- compaction / maintenance overhead

Only add indexes for real access paths.

## 8.1 Built-in indexes

Every collection has:

- a primary index on `_key`
- edge indexes for `_from` and `_to` on edge collections

## 8.2 Persistent indexes

Use for:

- equality filters
- range filters
- common sort orders
- multi-field lookup patterns
- most regular secondary indexing needs

Example:

```ts
await collection.ensureIndex({
  type: "persistent",
  name: "idx_brand_name",
  fields: ["name"],
});
```

### Use persistent indexes when:

- queries filter by exact values
- queries filter by ranges on one or more fields
- you need a compound index for common query prefixes
- you sort on indexed fields in compatible order

## 8.3 Unique persistent indexes

Use when business rules require uniqueness.

Example:

```ts
await collection.ensureIndex({
  type: "persistent",
  name: "uniq_brand_name",
  fields: ["name"],
  unique: true,
});
```

## 8.4 TTL indexes

Use only when documents should expire automatically.

Example:

```ts
await collection.ensureIndex({
  type: "ttl",
  name: "ttl_sessions_expiresAt",
  fields: ["expiresAt"],
  expireAfter: 0,
});
```

Use TTL for:

- sessions
- temporary tokens
- short-lived jobs
- caches

Avoid TTL when:

- deletion needs domain logic
- expiry rules are conditional or user-visible

## 8.5 Geo indexes

Use for:

- proximity search
- radius search
- geo containment / spatial workloads

Example:

```ts
await collection.ensureIndex({
  type: "geo",
  name: "geo_locations",
  fields: ["location"],
  geoJson: true,
});
```

## 8.6 Inverted indexes

Use for:

- fielded text search
- faceting
- filtered search
- ranking
- analyzers
- document retrieval through search expressions

These are central for search-style workloads.

## 8.7 Vector indexes

Use for embedding similarity search.
This feature is version-sensitive and may require a server startup option depending on server version and deployment.

Use vector indexes for:

- semantic search
- nearest-neighbor retrieval
- recommendation systems
- retrieval-augmented workloads

## 8.8 Vertex-centric indexes

Use on edge collections when traversals repeatedly filter by edge attributes in addition to `_from` or `_to`.

Example use case:

- edges have `type`
- traversal filters for `type == "friend"`

Pattern:

- create a persistent or prefixed multi-dimensional index with `_from` or `_to` plus the filtered edge attributes

## 8.9 Index design checklist

Before creating an index, ask:

- which `FILTER` uses it?
- is the query frequent?
- how selective is the predicate?
- does the same index support sorting too?
- will the write overhead be worth it?
- can one compound index replace two weaker ones?

---

# 9. Search patterns

## 9.1 When `LIKE` is enough

Use ordinary AQL `LIKE` for:

- small datasets
- admin search tools
- occasional substring lookups
- non-ranking requirements

Example:

```aql
FOR doc IN products
  FILTER LIKE(LOWER(doc.title), CONCAT('%', LOWER(@q), '%'))
  RETURN doc.title
```

## 9.2 When to use ArangoSearch / inverted indexes

Use search indexing when you need:

- relevance ranking
- analyzers
- tokenization
- wildcard / fuzzy / phrase behavior
- faceting
- complex search expressions
- larger search-heavy datasets

## 9.3 Search example

```aql
FOR doc IN productSearch
  SEARCH ANALYZER(doc.title IN TOKENS(@q, "text_en"), "text_en")
  SORT BM25(doc) DESC
  LIMIT 20
  RETURN {
    id: doc._id,
    title: doc.title,
    score: BM25(doc)
  }
```

## 9.4 Exact-match search pattern

```aql
FOR doc IN myView
  SEARCH ANALYZER(doc.title == @title, "identity")
  RETURN doc.title
```

## 9.5 Fuzzy / approximate search

Use analyzer-backed search or n-gram / fuzzy patterns when typo tolerance matters.
Do not fake fuzzy search with brute-force full scans in production.

---

# 10. Graph modeling and queries

## 10.1 When to use graph features

Use edge collections and traversals when:

- relation depth matters
- paths matter
- branching exploration matters
- you need shortest path / k-shortest paths / all shortest paths
- adjacency queries are first-class

## 10.2 Basic traversal

```aql
FOR v, e, p IN 1..3 OUTBOUND @startVertex brandRelations
  RETURN {
    vertex: v,
    edge: e,
    depth: LENGTH(p.edges)
  }
```

## 10.3 Traversal with filter

```aql
FOR v, e, p IN 1..3 OUTBOUND @startVertex brandRelations
  FILTER e.type == "owner"
  RETURN v
```

## 10.4 Traversal with PRUNE

Use `PRUNE` to stop exploring deeper when a path should not continue.

```aql
FOR v, e, p IN 1..5 OUTBOUND @startVertex brandRelations
  PRUNE e.status == "inactive"
  RETURN {
    vertexId: v._id,
    depth: LENGTH(p.edges)
  }
```

### Important distinction

- `FILTER` removes results
- `PRUNE` stops further exploration of that path

If the goal is traversal efficiency, `PRUNE` is often the right tool.

## 10.5 Shortest path

```aql
FOR v, e IN OUTBOUND SHORTEST_PATH @start TO @target brandRelations
  RETURN { vertex: v, edge: e }
```

## 10.6 k-shortest paths

```aql
FOR path IN OUTBOUND K_SHORTEST_PATHS @start TO @target brandRelations
  LIMIT 5
  RETURN path
```

## 10.7 Weighted traversal

Use weighted traversals or path queries only when edge weights truly matter.
Keep edge weight semantics explicit and consistent.

## 10.8 Graph design checklist

- Are relationships queried beyond one hop?
- Do relationships need their own attributes?
- Are traversals bounded?
- Is fan-out controlled?
- Do edge filters need vertex-centric indexing?
- Would embedding be simpler?

---

# 11. Transactions and consistency

## 11.1 Practical view

Transactions matter when:

- multiple documents must change atomically
- multiple collections must stay consistent
- read-modify-write logic must not observe partial state

## 11.2 Transaction types

At development time, think in two broad categories:

- **server-side / JavaScript transactions**
- **streaming transactions**

Streaming transactions are often the practical choice for application code using a driver because they support begin / step / commit / abort behavior across multiple requests.

## 11.3 `arangojs` streaming transaction example

```ts
import { Database, aql } from "arangojs";

const db = new Database();
const brands = db.collection("product_brands");
const audits = db.collection("audit_logs");

const trx = await db.beginTransaction({
  write: [brands, audits],
});

try {
  const cursor = await trx.step(() =>
    db.query(aql`
    FOR brand IN ${brands}
      FILTER brand._key == ${"brand-001"}
      UPDATE brand WITH {
        discount: ${15},
        updatedAt: DATE_ISO8601(DATE_NOW())
      } IN ${brands}
      RETURN NEW
  `),
  );
  const updated = await cursor.all();

  await trx.step(() =>
    audits.save({
      entityType: "brand",
      entityKey: "brand-001",
      action: "discount-updated",
      createdAt: new Date().toISOString(),
    }),
  );

  await trx.commit();
} catch (err) {
  await trx.abort();
  throw err;
}
```

## 11.4 Transaction guidance

Use a transaction when:

- multiple writes must succeed or fail together
- counters and related state must remain aligned
- you create entity + audit record + relation together

Avoid oversized transactions:

- large transactions consume memory
- large bulk writes may need chunking
- split work if atomicity is not required at the entire batch size

## 11.5 Isolation mindset

For most application work:

- assume snapshot-style reads within a transaction
- do not rely on side effects from concurrent transactions becoming visible mid-transaction
- keep transaction scope narrow and explicit

---

# 12. Schema validation

Schema validation is useful for:

- preventing malformed writes
- enforcing required fields
- protecting long-lived collections from drift

Typical use cases:

- DTO enforcement at the database level
- migration safety
- hardening multi-writer systems

## 12.1 Example schema

```json
{
  "rule": {
    "type": "object",
    "properties": {
      "name": { "type": "string", "minLength": 1 },
      "discount": { "type": "number" },
      "isDeleted": { "type": "boolean" }
    },
    "required": ["name", "isDeleted"],
    "additionalProperties": true
  },
  "level": "strict",
  "message": "Invalid brand document"
}
```

## 12.2 Validation levels

Common levels include:

- `none`
- `new`
- `moderate`
- `strict`

Choose `strict` when all new and modified documents must comply.
Choose `moderate` when legacy invalid data exists and you need a gradual cleanup strategy.

## 12.3 Schema guidance

- validate core invariants in DB, not only app code
- do not overfit schema to transient API shapes
- use clear error messages
- pair schema with migrations

---

# 13. Computed values

Computed values let a collection derive top-level attributes automatically on insert, update, or both.

Use computed values for:

- normalized search fields
- derived denormalized fields
- write-time convenience fields

Examples:

- `nameLower`
- `fullName`
- `sortableTitle`
- `updatedAtDateOnly`

### Good uses

- normalize casing
- precompute a stable derived field that many queries use
- reduce repeated application logic

### Avoid

- complex business logic
- derived values that are difficult to reason about
- anything better handled as explicit application behavior

---

# 14. `arangojs` essentials

## 14.1 Basic setup

```ts
import { Database } from "arangojs";

const db = new Database({
  url: process.env.ARANGODB_URL,
});

db.useDatabase(process.env.ARANGODB_DATABASE!);
db.useBasicAuth(process.env.ARANGODB_USERNAME!, process.env.ARANGODB_PASSWORD!);
```

## 14.2 Collections

```ts
const brands = db.collection("product_brands");
const users = db.collection("users");
const brandRelations = db.edgeCollection("brand_relations");
```

## 14.3 Safe dynamic AQL with `aql`

Always use `aql` for dynamic queries.

```ts
import { aql } from "arangojs";

const cursor = await db.query(aql`
  FOR brand IN ${brands}
    RETURN brand
`);
```

## 14.4 Compose optional clauses with `join`

```ts
import { aql } from "arangojs";
import { join } from "arangojs/aql";

const filters = [aql`FILTER brand.isDeleted == false`];

if (q?.trim()) {
  filters.push(
    aql`FILTER LIKE(LOWER(brand.name), CONCAT('%', LOWER(${q.trim()}), '%'))`,
  );
}

const query = aql`
  FOR brand IN ${brands}
    ${join(filters, "\n")}
    RETURN brand
`;
```

## 14.5 Never inject raw query fragments as strings

Bad:

```ts
const filter = `FILTER brand.name == "${q}"`;

const query = aql`
  FOR brand IN ${brands}
    ${filter}
    RETURN brand
`;
```

This fails because the string becomes a bind parameter, not actual query syntax.

## 14.6 `literal(...)`

Use `literal(...)` only for rare trusted edge cases.
Prefer nested `aql` fragments almost always.

## 14.7 Dynamic sorting pattern

```ts
const sortFieldMap: Record<string, any> = {
  createdAt: aql`brand.createdAt`,
  name: aql`brand.name`,
  discount: aql`brand.discount`,
};

const sortField = sort
  ? (sortFieldMap[sort] ?? aql`brand.createdAt`)
  : aql`brand.createdAt`;

const sortDirection = order === "ASC" ? aql`ASC` : aql`DESC`;
```

## 14.8 Pagination pattern

```ts
const offset = (currentPage - 1) * limit;
const limitClause = fetchAll ? aql`` : aql`LIMIT ${offset}, ${limit}`;
```

## 14.9 Full repository `find()` pattern

```ts
async find(dto, options) {
  const {
    q,
    limit = 20,
    currentPage = 1,
    sort,
    order = "DESC",
    fetchAll = false,
  } = dto;

  const filters = [
    aql`FILTER brand.isDeleted == false`,
  ];

  if (q?.trim()) {
    filters.push(
      aql`FILTER LIKE(LOWER(brand.name), CONCAT('%', LOWER(${q.trim()}), '%'))`
    );
  }

  const sortFieldMap = {
    createdAt: aql`brand.createdAt`,
    name: aql`brand.name`,
    discount: aql`brand.discount`,
  };

  const sortField =
    sort ? sortFieldMap[sort] ?? aql`brand.createdAt` : aql`brand.createdAt`;
  const sortDirection = order === "ASC" ? aql`ASC` : aql`DESC`;

  const offset = (currentPage - 1) * limit;
  const limitClause = fetchAll ? aql`` : aql`LIMIT ${offset}, ${limit}`;

  const query = aql`
    LET total = FIRST(
      FOR brand IN ${this.brandsCollection}
        ${join(filters, "\n")}
        COLLECT WITH COUNT INTO count
        RETURN count
    )

    LET rows = (
      FOR brand IN ${this.brandsCollection}
        ${join(filters, "\n")}
        SORT ${sortField} ${sortDirection}
        ${limitClause}
        RETURN {
          id: brand._key,
          name: brand.name,
          discount: brand.discount
        }
    )

    RETURN { total, rows }
  `;

  const [result] = await this.runAql(query);

  return {
    data: result?.rows ?? [],
    meta: {
      total: result?.total ?? 0,
      limit,
      currentPage,
    },
  };
}
```

---

# 15. Query performance and diagnostics

## 15.1 First questions to ask

When a query is slow, ask:

- is it scanning too much data?
- is an index missing?
- is the result shape too large?
- are the most selective filters applied early enough?
- is sorting forcing extra work?
- is search being done with `LIKE` when search indexing is needed?
- is a graph traversal exploring too much?
- can `PRUNE` stop useless paths early?

## 15.2 Use `EXPLAIN`

Use `EXPLAIN` when you need to know:

- whether an index is used
- what optimizer rules fired
- whether filtering is early or late
- how traversal nodes are planned

## 15.3 Use profiling

Profile important queries to inspect:

- scanned documents
- index usage
- filtered documents
- runtime
- memory
- cache hits / misses where applicable

## 15.4 Plan cache

Plan caching can reduce repeated planning overhead for identical query shapes.
Do not confuse:

- **query result cache**
- **query plan cache**

They solve different problems.

## 15.5 Result cache

Use result caching carefully and only when:

- results are reused
- the data does not change too often
- correctness expectations are clear

## 15.6 Query shape tips

### Good

```aql
FOR doc IN coll
  FILTER doc.isDeleted == false
  FILTER doc.status == @status
  SORT doc.createdAt DESC
  LIMIT 20
  RETURN {
    id: doc._key,
    status: doc.status,
    createdAt: doc.createdAt
  }
```

### Bad

```aql
FOR doc IN coll
  RETURN doc
```

Then filter and sort in application code.

## 15.7 Bulk update performance

For large update-style AQL jobs, consider:

- chunking
- exclusive query option when appropriate
- intermediate commits through API options when atomicity of the entire full batch is not required

---

# 16. Common patterns

## 16.1 Generic list endpoint

```aql
FOR doc IN coll
  FILTER doc.isDeleted == false
  SORT doc.createdAt DESC
  LIMIT @offset, @limit
  RETURN KEEP(doc, "_key", "name", "createdAt")
```

## 16.2 Uniqueness check before insert

Prefer database uniqueness when possible.
If app-level prechecks are still needed:

```aql
RETURN LENGTH(
  FOR doc IN coll
      FILTER LOWER(doc.name) == LOWER(@name)
    LIMIT 1
    RETURN 1
) > 0
```

## 16.3 Ownership enrichment

```aql
FOR brand IN product_brands
    LET owner = FIRST(
    FOR user IN users
      FILTER user._id == brand.ownerId
      RETURN KEEP(user, "_key", "firstName", "lastName", "email")
  )
  RETURN MERGE(
    KEEP(brand, "_key", "name", "discount"),
    { owner }
  )
```

## 16.4 Edge existence

```aql
RETURN LENGTH(
  FOR edge IN brand_relations
    FILTER edge._from == @from
    FILTER edge._to == @to
    LIMIT 1
    RETURN 1
) > 0
```

## 16.5 Audit write pattern

```aql
LET updated = FIRST(
  FOR doc IN brands
    FILTER doc._key == @id
    UPDATE doc WITH { discount: @discount } IN brands
    RETURN NEW
)

INSERT {
  entityType: "brand",
  entityKey: updated._key,
  action: "updated",
  createdAt: DATE_ISO8601(DATE_NOW())
} INTO audit_logs

RETURN updated
```

Wrap in a transaction if both writes must succeed or fail together.

## 16.6 Search endpoint escalation path

Use this rule:

1. start with `LIKE` for tiny/simple admin tools
2. move to search indexing when search becomes product-facing or large-scale
3. move to vector search when semantic similarity is the requirement

---

# 17. Anti-patterns

## 17.1 Raw string AQL composition

```ts
const sort = `SORT doc.${field}`;
```

Do not do this.

## 17.2 Missing required scope filters

If the application uses workspace, organization, project, or account scoping, every relevant query must apply that boundary consistently.

## 17.3 Over-indexing

Adding many overlapping indexes slows writes and increases memory use.

## 17.4 Deep graph traversals without bounds

Always bound depth unless the domain truly requires otherwise.

## 17.5 Returning full documents in hot paths

Project the fields you need.

## 17.6 Using search features for exact key lookups

Use `_key`, `_id`, or persistent indexes instead.

## 17.7 Using traversal when a single join is enough

Sometimes one subquery is simpler and faster than a graph traversal.

---

# 18. Troubleshooting

## Error: `unexpected bind parameter near '@value2'`

Cause:

- a raw string containing AQL syntax was interpolated into an `aql` template

Bad:

```ts
const filter = "FILTER x.a == 1";
aql`FOR x IN ${coll} ${filter} RETURN x`;
```

Fix:

```ts
const filter = aql`FILTER x.a == ${1}`;
aql`FOR x IN ${coll} ${filter} RETURN x`;
```

## Error: query returns no rows

Check:

- wrong scope filter
- soft-delete filter excludes rows
- `_key` vs `_id` mismatch
- `LIMIT` offset too large
- case sensitivity
- unexpected nulls

## Error: sorting fails or is unsafe

Check:

- dynamic sort field not whitelisted
- invalid field name
- raw string injection
- sort direction not mapped to fixed fragments

## Error: query is slow

Check:

- missing persistent index
- too much result projection
- scan instead of indexed access
- `LIKE` on large data without search indexing
- traversal exploring too many paths
- no `PRUNE`
- count query and result query duplicated inefficiently

## Error: transaction too large or memory-heavy

Check:

- batch size
- number of modified docs
- payload size
- whether the whole job truly needs one transaction

## Error: schema rejects writes

Check:

- required fields
- field types
- validation level
- mismatch between DTO and stored shape

## Error: search relevance looks wrong

Check:

- analyzer choice
- identity vs text analyzer
- tokenization expectations
- whether exact matching or ranking is actually needed

---

# 19. Review checklist for the agent

Before finalizing a solution, verify:

## Model

- Does the data model fit the access pattern?
- Should a relation be embedded, referenced, or edged?

## Query safety

- Are values parameterized?
- Is query syntax composed with `aql`, not strings?
- Are dynamic field names whitelisted?

## Query correctness

- Are required filters such as scope, status, or soft-delete present where needed?
- Is `_key` vs `_id` handled correctly?
- Does the query have a `RETURN`?

## Query performance

- Is an index likely needed?
- Is the result shape narrow enough?
- Is the sort compatible with the index strategy?
- Is traversal depth bounded?
- Can `PRUNE` help?

## Write safety

- Should this be transactional?
- Should uniqueness be enforced in DB?
- Does schema validation need to be updated?

---

# 20. How the agent should answer users

When the user asks about ArangoDB:

1. Identify whether the problem is mainly:
   - data modeling
   - query syntax
   - dynamic query composition
   - indexing
   - graph traversal
   - search
   - transactions
   - driver usage
   - performance
2. Explain the actual failure mode precisely.
3. Give the smallest working fix first.
4. Then give the safer production-ready version.
5. Mention the governing rule in one sentence.

### Good answer pattern

- **Problem**
- **Why it fails**
- **Minimal fix**
- **Production-safe version**
- **Rule to remember**

---

# 21. High-value snippets

## Safe equality filter

```ts
aql`FILTER doc.status == ${status}`;
```

## Safe contains search

```ts
aql`FILTER LIKE(LOWER(doc.name), CONCAT('%', LOWER(${q}), '%'))`;
```

## Safe limit

```ts
aql`LIMIT ${offset}, ${limit}`;
```

## Safe sort direction

```ts
const direction = order === "ASC" ? aql`ASC` : aql`DESC`;
```

## Safe count

```aql
COLLECT WITH COUNT INTO count
RETURN count
```

## Safe keep projection

```aql
RETURN KEEP(doc, "_key", "name", "createdAt")
```

---

# 22. Development heuristics

Use these rules as defaults:

- **Documents first, graph when relationships become first-class**
- **Persistent indexes first, search indexes when search semantics matter**
- **AQL for most application logic**
- **Transactions for multi-write consistency**
- **Schema validation for durable invariants**
- **Computed values for stable derived fields**
- **Profile before “optimizing”**
- **Bind values, compose syntax**

---

# 23. Source map for further verification

The agent should prefer the official references below when checking details:

- ArangoDB stable docs:
  - AQL
  - Examples and query patterns
  - Execution and performance
  - Indexes and search
  - Graph queries
  - Transactions
  - Data models
  - Operational factors
  - Server options
- arangojs latest docs:
  - `Database`
  - `DocumentCollection`
  - `aql`
  - `literal`
  - transactions

---

# 24. One-line summary

**In ArangoDB development: model around access patterns, bind values, compose AQL safely, index only real workloads, and profile the hot paths.**
