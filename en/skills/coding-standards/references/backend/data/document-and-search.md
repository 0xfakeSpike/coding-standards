# Document databases and search

This document covers MongoDB and Elasticsearch. For relational databases, see [schema-design.md](schema-design.md).

## Selection boundaries

### Default to a relational database [Recommended]

Start with a relational database. Add other storage only when the needs below are clear. Each additional store adds another set of operations, backups, and syncing.

| Need | Choice |
|---|---|
| Transactions, relational queries, stably structured data | MySQL, PostgreSQL |
| Data with widely varying structure, read and written as a whole, accessed by aggregate | MongoDB |
| Full-text search, relevance ranking, multi-dimensional filtering and aggregation | Elasticsearch |

### Do not use Elasticsearch as the source of truth [Required]

Data in Elasticsearch is an index derived from the primary store. If the primary store is lost, business data cannot be recovered from it. Business decisions read the primary store, not search results.

**Why:** Syncing has a delay. A write can take up to about 1 second to become searchable, and the gap grows when syncing fails.

**Source:** Elasticsearch Docs

## MongoDB modeling

### Design documents around access patterns [Recommended]

List the most frequent reads and writes first, then decide the document structure. Keep data that is accessed together in one place. Do not copy the relational table structure.

**Source:** MongoDB Docs

### Embed by default, reference for frequent updates [Recommended]

| Situation | Choice |
|---|---|
| Containment, child data always read with the parent | Embed |
| One-to-many, with a bounded number of children | Embed |
| Related data updated often, or shared in several places | Reference |
| Unbounded number of children | Reference |

Embedding lets you read with one operation and update in one atomic write. Referencing needs only one update, but reading takes an extra query or `$lookup`.

**Source:** MongoDB Docs

### Do not use unbounded arrays [Required]

When array elements keep growing, move them to a separate collection, or embed only a small number of the most-used elements.

**Why:** A single document must be smaller than 16 MiB. Unbounded arrays also slow down indexes and the application.

**Example:**

Avoid:

```json
{ "title": "Product A", "reviews": [ { "user": "u1", "text": "..." } ] }
```

Prefer:

```json
{ "title": "Product A", "recent_reviews": [ { "user": "u1", "text": "..." } ] }
```

```json
{ "product_id": "p1", "user": "u1", "text": "..." }
```

The second document goes in the `reviews` collection, and the product document keeps only the latest few.

**Source:** MongoDB Docs

### Have a sync plan for duplicated data [Recommended]

Only data that rarely changes is suitable for duplicate storage. Before duplicating, decide whether immediate consistency is needed, and whether to sync with a transaction or asynchronously.

**Source:** MongoDB Docs

### Enable validation once the structure is stable [Recommended]

MongoDB does not constrain document structure by default. Once the application's structure is settled, use `$jsonSchema` to validate required fields, types, and value ranges, so bad data cannot be written.

**Source:** MongoDB Docs

### Prefer document structure for atomicity [Recommended]

Single-document operations are atomic. Putting related data in one document reduces the need for multi-document transactions. When a multi-document transaction is required, keep it short. By default it cannot run longer than one minute, and it is aborted on timeout.

**Source:** MongoDB Docs

### Follow ESR for compound indexes [Recommended]

Put equality fields first. To avoid in-memory sorts, put sort fields before range fields. When a range condition is highly selective, put the range field before the sort field.

**Source:** MongoDB Docs

## Elasticsearch mappings

### Use explicit mappings for production indexes [Recommended]

**Applies to:** production service

Define types for known fields explicitly when creating the index. Do not rely on dynamic mapping inference. Use `dynamic` to control unknown fields: `strict` rejects the document, and `false` leaves new fields unindexed.

**Why:** Inferred types may not be what you want, and mappings of existing fields usually cannot be changed.

**Example:**

```json
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "sku": { "type": "keyword" },
      "title": { "type": "text" },
      "price": { "type": "scaled_float", "scaling_factor": 100 },
      "created_at": { "type": "date" }
    }
  }
}
```

**Source:** Elasticsearch Docs

### Distinguish text and keyword [Recommended]

| Type | Use |
|---|---|
| `text` | Full-text search after analysis, such as titles and body text |
| `keyword` | Exact match, sorting, and aggregation, such as status, tags, and email |

Use `keyword` for numeric identifiers that do not need range queries. `term` queries are usually faster. When both uses are needed, use a multi-field mapping.

**Source:** Elasticsearch Docs

### Prevent mapping explosion [Required]

**Applies to:** production service

Do not use user input or dynamic values as field names. Use the `flattened` type for objects with many non-fixed keys. Keep the `index.mapping.total_fields.limit` limit, and do not solve problems by raising it.

**Why:** Too many fields cause out-of-memory errors, and the cluster becomes hard to recover.

**Source:** Elasticsearch Docs

### Put index configuration in templates [Recommended]

Define mappings and settings with index templates, extract shared parts into component templates, and keep them under version control. Templates apply only when an index is created. Changing a template does not affect existing indexes.

**Source:** Elasticsearch Docs

### Access indexes through aliases [Recommended]

Applications use only aliases. To change a mapping, create a new index, import the data with `_reindex`, then switch atomically with a single alias operation.

**Example:**

```json
{
  "actions": [
    { "remove": { "index": "products_20261001", "alias": "products" } },
    { "add": { "index": "products_20261105", "alias": "products" } }
  ]
}
```

**Source:** Elasticsearch Docs

## Queries and sync

### Use search_after for deep pagination [Required]

`from` plus `size` cannot exceed 10000 by default, and deeper pages use more memory. For deep pagination, use `search_after` with a point in time (PIT), and delete the PIT when done. Do not use `scroll` for deep pagination.

**Source:** Elasticsearch Docs

### Do not depend on search results right after a write [Required]

Writes become searchable only after a refresh. Do not solve this by forcing a refresh on every write. Pages that display data right after a write read the primary store.

**Source:** Elasticsearch Docs

### Sync to the search index asynchronously [Recommended]

Do not write to both stores synchronously in a business request. After the primary store commits, update the index through messages or change data capture. For delivery reliability, see [consistency.md](../reliability/consistency.md). For consumption rules, see [messaging.md](../async/messaging.md).

The sync must meet these requirements:

- Use the primary key as the document `_id`, so repeated writes do not change the result.
- Use an increasing external version (`version_type=external`) to keep out-of-order old data from overwriting new data.
- Be able to rebuild the full index from the primary store. This is also the repair method when data becomes inconsistent.
