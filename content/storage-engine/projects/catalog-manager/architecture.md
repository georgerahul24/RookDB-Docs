---
title: Architecture
sidebar_position: 2
---

# Catalog Manager — Architecture

## Catalog Storage Architecture

The Catalog Manager introduces a **page-based storage backend** that stores system catalog metadata in dedicated `.dat` files under `database/global/catalog_pages/`. Each catalog file follows the same slotted-page format used by user tables, enabling seamless integration with the buffer manager.

### Directory Layout

```
database/
├── global/
│   ├── catalog_pages/              # Page-based catalog storage
│   │   ├── pg_database.dat         # System catalog: databases
│   │   ├── pg_table.dat            # System catalog: tables
│   │   ├── pg_column.dat           # System catalog: columns
│   │   ├── pg_constraint.dat       # System catalog: constraints
│   │   ├── pg_index.dat            # System catalog: indexes
│   │   └── pg_type.dat             # System catalog: data types
│   ├── pg_oid_counter.dat          # Persistent OID counter
│   └── catalog.json                # DEPRECATED: Legacy format
└── base/
    └── {database}/
        ├── {table}.dat             # User table data files
        └── indexes/                # Index files
            └── {index_name}.idx
```

### Design Rationale

| Aspect | Rationale |
|--------|-----------|
| Page-based storage | Enables buffer manager integration for efficient caching |
| System catalogs | Provides self-hosting capabilities similar to PostgreSQL |
| Separation of concerns | System metadata is cleanly separated from user data |
| Scalability | Supports large numbers of databases, tables, and constraints |

---

## Page Layout for Catalog Pages

Catalog pages use the **identical slotted-page layout** as user table files, consistent with RookDB's existing page structure:

- **Page 0** — Table header (8,192 bytes; first 4 bytes = total page count)
- **Page 1+** — Slotted data pages (8,192 bytes each)

Each data page consists of:
- A **page header** (lower and upper pointers, 8 bytes)
- An **Item ID array** growing forward from the header
- **Tuple data** appended from the end of the page backward

> **Implementation Note:** Catalog files are initialised using the same `init_table()` function as user tables. The original design document did not specify the page-0 length, leading to the discovery that a short header would break all seeks in the disk manager (see [Implementation Notes](./implementation-notes.md) §1).

---

## Buffer Manager Integration

All catalog page I/O is routed through the existing Buffer Manager using `pin_page()` / `unpin_page()` semantics:

- **`CatalogPageManager`** maps catalog names to file paths and delegates all reads/writes to the buffer pool.
- **Pin/unpin** semantics prevent eviction of actively used catalog pages.
- **Dirty tracking** ensures modified catalog pages are written back to disk.
- **LRU replacement** maximises cache hit rate for frequently accessed catalogs.

The integration points are:

```rust
// Every CRUD operation uses the buffer manager
pm.insert_catalog_tuple(bm, CAT_TABLE, bytes)?;
pm.scan_catalog(bm, CAT_DATABASE)?;
pm.delete_catalog_tuple(bm, CAT_INDEX, page_num, slot_id)?;
```

---

## OID (Object Identifier) System

Every database object is assigned a globally unique **32-bit Object Identifier (OID)**. OIDs enable referential integrity across the system catalog tables.

### OID Ranges

| Range | Purpose |
|-------|---------|
| `1 – 9,999` | Reserved for built-in types and system objects |
| `10,000+` | User-created objects (databases, tables, columns, etc.) |

### Persistence

The next available OID is stored as a little-endian `u32` in `database/global/pg_oid_counter.dat`:

- On startup, the counter is loaded from this file.
- When the page backend is active, every `alloc_oid()` call writes the incremented counter directly to the file, preventing OID reuse after a crash.
- In legacy JSON mode, the counter is captured implicitly inside `catalog.json`.

### Allocation

```rust
pub fn alloc_oid(&mut self) -> u32 {
    let oid = self.oid_counter;
    self.oid_counter += 1;
    if self.page_backend_active {
        // Write to pg_oid_counter.dat immediately
    }
    oid
}
```

---

## Catalog Cache

The in-memory **LRU Catalog Cache** reduces disk I/O for frequently accessed metadata:

### Cache Entries

| Entry Type | Key | Value |
|------------|-----|-------|
| Database | `db_name` | `Database` struct |
| Table | `(db_oid, table_name)` | `Table` struct |
| Constraints | `table_oid` | `Vec<Constraint>` |
| Indexes | `table_oid` | `Vec<Index>` |
| Types | `type_oid` | `DataType` struct |

### Cache Policy

- **Max size:** 256 entries (configurable)
- **Eviction:** LRU (Least Recently Used) — when capacity is reached, the oldest entry is removed
- **Invalidation:** Every DDL operation (CREATE, ALTER, DROP) eagerly invalidates affected cache entries
- **Write-through:** Changes are always persisted to pages first; the cache is populated lazily on reads

### Invalidation Points

| Operation | Invalidation |
|-----------|-------------|
| `create_database` | `invalidate_database(db_name)` |
| `drop_database` | `invalidate_database(db_name)` |
| `create_table` | `invalidate_table(db_oid, table_name)` |
| `drop_table` | `invalidate_table`, `invalidate_constraints`, `invalidate_indexes` |
| `alter_table_add_column` | `invalidate_constraints(table_oid)` |
| `add_*_constraint` | `invalidate_constraints(table_oid)` |
| `create_index` / `drop_index` | `invalidate_indexes(table_oid)` |

---

## Dual-Mode Initialization

The catalog system supports two storage backends for migration compatibility:

1. **Page mode** — page-based storage under `database/global/catalog_pages/`
2. **Legacy mode** — JSON-based `database/global/catalog.json`

### Startup Flow

```
init_catalog(bm)
  │
  ├── catalog_pages/ exists?
  │     └── YES → Page backend detected (load from pages)
  │
  └── NO → Bootstrap
            ├── Create catalog_pages/ directory
            ├── Initialize all 6 system catalog files
            ├── Register built-in types in pg_type
            └── Create "system" database record in pg_database
```

### Bootstrap

On a fresh install, `bootstrap_catalog()`:

1. Creates the `database/global/catalog_pages/` directory
2. Initialises the OID counter at `10,000`
3. Creates all six system catalog `.dat` files using `init_table()`
4. Registers all 10 built-in data types into `pg_type`
5. Inserts the system database record (`db_oid=1`, `name="system"`) into `pg_database`
