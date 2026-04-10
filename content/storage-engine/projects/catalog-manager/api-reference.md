---
title: API Reference
sidebar_position: 5
---

# Catalog Manager — API Reference

All public functions in the Catalog Manager are documented below, grouped by subsystem. Functions are defined in `src/backend/catalog/catalog.rs`, `constraints.rs`, and `indexes.rs`.

---

## Catalog Initialization

### `init_catalog`

**Description:**  
Dual-mode catalog initialisation, called at startup. Detects whether page-based catalog storage exists and bootstraps if necessary.

**Function:**
```rust
pub fn init_catalog(bm: &mut BufferManager)
```

**Input:**
- `bm` — Mutable reference to the buffer manager.

**Implementation:**
1. Create `database/global/` and `database/base/` directories if they do not exist.
2. If `database/global/catalog_pages/` exists, report that the page backend is detected.
3. Otherwise, call `bootstrap_catalog(bm)` to initialise the system from scratch.

---

### `bootstrap_catalog`

**Description:**  
Bootstrap the self-hosting catalog: creates system catalog `.dat` files, inserts built-in types, and writes the system database record.

**Function:**
```rust
pub fn bootstrap_catalog(bm: &mut BufferManager) -> Result<(), CatalogError>
```

**Implementation:**
1. Ensure `database/global/` and `database/base/` directories exist.
2. Initialise the OID counter via `OidCounter::initialize()`.
3. Create a new `CatalogPageManager` and initialise all six system catalog files.
4. Register all 10 built-in data types into `pg_type`.
5. Insert the system database record (`db_oid=1`, `name="system"`) into `pg_database`.

---

### `init_catalog_page_storage`

**Description:**  
Create or verify the `CatalogPageManager` after bootstrap.

**Function:**
```rust
pub fn init_catalog_page_storage() -> Result<CatalogPageManager, CatalogError>
```

**Output:**
- Returns a fully initialised `CatalogPageManager` with all file paths registered.

---

## Load / Save

### `load_catalog`

**Description:**  
Load the Catalog from the active storage backend. Attempts page-based loading first; falls back to an empty catalog.

**Function:**
```rust
pub fn load_catalog(bm: &mut BufferManager) -> Catalog
```

**Output:**
- Returns a `Catalog` struct populated with all databases, tables, columns, constraints, and index OIDs from the page backend.

**Implementation:**
1. If `catalog_pages/` exists, load from pages via `load_catalog_from_pages(bm)`.
2. On failure, return `Catalog::new()` (empty catalog).

The page-based loader:
1. Initialises `OidCounter` from `pg_oid_counter.dat`.
2. Scans `pg_database` → populates `catalog.databases`.
3. Scans `pg_table` → attached to parent databases by `db_oid`.
4. Scans `pg_column` → attached to parent tables by `table_oid`, sorted by `column_position`.
5. Scans `pg_constraint` → attached to parent tables by `table_oid`.
6. Scans `pg_index` → index OIDs attached to parent tables by `table_oid`.

---

## Type Helpers

### `register_builtin_types`

**Description:**  
Register all built-in data types in `pg_type`. Skips types that already exist.

**Function:**
```rust
pub fn register_builtin_types(
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
) -> Result<(), CatalogError>
```

---

### `lookup_type_by_name`

**Description:**  
Resolve a type name to a `DataType` struct. Checks the built-in type list first, then scans `pg_type`.

**Function:**
```rust
pub fn lookup_type_by_name(
    pm: &CatalogPageManager,
    bm: &mut BufferManager,
    type_name: &str,
) -> Result<DataType, CatalogError>
```

**Output:**
- Returns the matching `DataType` on success, or `CatalogError::TypeNotFound` on failure.

---

## Database Operations

### `create_database`

**Description:**  
Create a new database with metadata (owner, encoding) and persist it to `pg_database`.

**Function:**
```rust
pub fn create_database(
    catalog: &mut Catalog,
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
    db_name: &str,
    owner: &str,
    encoding: Encoding,
) -> Result<u32, CatalogError>
```

**Input:**
- `db_name` — Name of the new database (must be non-empty and unique).
- `owner` — Owner string.
- `encoding` — Character encoding (`Encoding::UTF8` or `Encoding::ASCII`).

**Output:**
- Returns the allocated `db_oid` on success.

**Implementation:**
1. Validate that the name is non-empty and not already used.
2. Allocate a new OID via `catalog.alloc_oid()`.
3. Create the `database/base/{db_name}/` directory.
4. Serialise and insert a record into `pg_database`.
5. Add the `Database` struct to the in-memory catalog.
6. Invalidate the database cache entry.

---

### `drop_database`

**Description:**  
Drop a database and all its tables.

**Function:**
```rust
pub fn drop_database(
    catalog: &mut Catalog,
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
    db_name: &str,
) -> Result<(), CatalogError>
```

**Implementation:**
1. Resolve `db_oid` from the in-memory catalog.
2. Drop all tables belonging to this database via `drop_table()`.
3. Find and delete the database record from `pg_database`.
4. Remove the database directory from disk.
5. Remove from in-memory catalog and invalidate cache.

---

### `show_databases`

**Description:**  
Display all databases from the page-based catalog with metadata.

**Function:**
```rust
pub fn show_databases(
    catalog: &Catalog,
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
)
```

**Output:**
- Prints a formatted table: `Database | Owner | Created At`.

---

## Table Operations

### `create_table`

**Description:**  
Create a new table with columns and constraints, persisting to `pg_table` and `pg_column`.

**Function:**
```rust
pub fn create_table(
    catalog: &mut Catalog,
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
    db_name: &str,
    table_name: &str,
    col_defs: Vec<ColumnDefinition>,
    constraint_defs: Vec<ConstraintDefinition>,
) -> Result<u32, CatalogError>
```

**Input:**
- `col_defs` — Column definitions with type names, nullability, and defaults.
- `constraint_defs` — Constraint definitions (PK, FK, UNIQUE, NOT NULL).

**Output:**
- Returns the allocated `table_oid` on success.

**Implementation:**
1. Validate that the database exists and the table name is unique.
2. Allocate `table_oid` and OIDs for each column.
3. Resolve each column's type via `DataType::from_name()`.
4. Serialise and insert column records into `pg_column`.
5. Create the table data file (`{db_name}/{table_name}.dat`) and initialise it.
6. Serialise and insert a record into `pg_table`.
7. Add the `Table` to the in-memory catalog and invalidate cache.
8. Process each constraint definition (PK, FK, UNIQUE, NOT NULL) via the respective constraint functions.

---

### `drop_table`

**Description:**  
Drop a table and all dependent objects (indexes, constraints).

**Function:**
```rust
pub fn drop_table(
    catalog: &mut Catalog,
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
    table_oid: u32,
) -> Result<(), CatalogError>
```

**Implementation:**
1. Check for foreign key dependencies from other tables — return `ForeignKeyDependency` error if found.
2. Drop all indexes on this table via `drop_index()`.
3. Locate the table's database name and table name.
4. Remove the table data file from disk.
5. Delete the record from `pg_table`.
6. Remove from in-memory catalog and invalidate all related cache entries.

---

### `alter_table_add_column`

**Description:**  
Add a new column to an existing table.

**Function:**
```rust
pub fn alter_table_add_column(
    catalog: &mut Catalog,
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
    table_oid: u32,
    col_def: ColumnDefinition,
) -> Result<u32, CatalogError>
```

**Output:**
- Returns the allocated `column_oid` on success.

**Constraints:**
- If the column is `NOT NULL`, a default value **must** be provided (otherwise returns `InvalidOperation`).
- Column name must not already exist in the table.

---

### `show_tables`

**Description:**  
Display all tables in a database from the page-based catalog with statistics.

**Function:**
```rust
pub fn show_tables(
    catalog: &Catalog,
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
    db_name: &str,
)
```

**Output:**
- Prints a formatted table: `Table Name | Rows | Pages | Created At`.

---

### `get_table_metadata`

**Description:**  
Retrieve complete table metadata including resolved columns, constraints, and indexes.

**Function:**
```rust
pub fn get_table_metadata(
    catalog: &Catalog,
    pm: &CatalogPageManager,
    bm: &mut BufferManager,
    db_name: &str,
    table_name: &str,
) -> Result<TableMetadata, CatalogError>
```

**Output:**
- Returns a `TableMetadata` struct with full `Column`, `Constraint`, and `Index` data.

---

## Constraint Management

### `add_primary_key_constraint`

**Description:**  
Add a primary key constraint to a table. Automatically creates a backing unique B-Tree index and sets referenced columns to `NOT NULL`.

**Function:**
```rust
pub fn add_primary_key_constraint(
    catalog: &mut Catalog,
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
    table_oid: u32,
    column_names: Vec<String>,
    constraint_name: Option<String>,
) -> Result<u32, CatalogError>
```

**Output:**
- Returns the allocated `constraint_oid`.

**Errors:**
- `AlreadyHasPrimaryKey` if the table already has a primary key.

---

### `add_foreign_key_constraint`

**Description:**  
Add a foreign key constraint referencing another table's primary key or unique columns.

**Function:**
```rust
pub fn add_foreign_key_constraint(
    catalog: &mut Catalog,
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
    table_oid: u32,
    column_names: Vec<String>,
    referenced_table_oid: u32,
    referenced_column_names: Vec<String>,
    on_delete: ReferentialAction,
    on_update: ReferentialAction,
    constraint_name: Option<String>,
) -> Result<u32, CatalogError>
```

**Validations:**
- Column counts must match between referencing and referenced tables.
- Referenced columns must be covered by a `PRIMARY KEY` or `UNIQUE` constraint.

---

### `add_unique_constraint`

**Description:**  
Add a unique constraint to a table. Automatically creates a backing unique B-Tree index.

**Function:**
```rust
pub fn add_unique_constraint(
    catalog: &mut Catalog,
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
    table_oid: u32,
    column_names: Vec<String>,
    constraint_name: Option<String>,
) -> Result<u32, CatalogError>
```

---

### `add_not_null_constraint`

**Description:**  
Add a NOT NULL constraint to a column (sets `is_nullable = false`).

**Function:**
```rust
pub fn add_not_null_constraint(
    catalog: &mut Catalog,
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
    table_oid: u32,
    column_oid: u32,
) -> Result<(), CatalogError>
```

---

### `validate_constraints`

**Description:**  
Validate all constraints for a tuple before insertion. Called during data loading (e.g., CSV import).

**Function:**
```rust
pub fn validate_constraints(
    catalog: &Catalog,
    pm: &CatalogPageManager,
    bm: &mut BufferManager,
    table_oid: u32,
    tuple_values: &HashMap<u32, Option<Vec<u8>>>,
) -> Result<(), ConstraintViolation>
```

**Validation logic:**
- **NOT NULL:** Returns `NotNullViolation` if any non-nullable column has a `None` value.
- **PRIMARY KEY / UNIQUE:** Checks the backing B-Tree index for duplicates via `index_lookup()`.
- **FOREIGN KEY:** Verifies referenced values exist in the referenced table's index.

---

### `get_constraints_for_table`

**Description:**  
Get all constraints for a table by scanning `pg_constraint`.

**Function:**
```rust
pub fn get_constraints_for_table(
    catalog: &Catalog,
    pm: &CatalogPageManager,
    bm: &mut BufferManager,
    table_oid: u32,
) -> Result<Vec<Constraint>, CatalogError>
```

---

## Index Management

### `create_index`

**Description:**  
Create a B-Tree index on specified columns.

**Function:**
```rust
pub fn create_index(
    catalog: &mut Catalog,
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
    table_oid: u32,
    column_oids: Vec<u32>,
    is_unique: bool,
    is_primary: bool,
    index_name: Option<String>,
) -> Result<u32, CatalogError>
```

**Implementation:**
1. Resolve the database name for the table.
2. Generate an index name if not provided (`idx_{table_oid}_{col_oids}`).
3. Create the indexes directory and `.idx` file with an initialised B-Tree root page.
4. Allocate an `index_oid` and persist the index record to `pg_index`.
5. Add the index OID to the table's in-memory `indexes` list.

---

### `drop_index`

**Description:**  
Drop an index by its OID, removing the index file and catalog entry.

**Function:**
```rust
pub fn drop_index(
    catalog: &mut Catalog,
    pm: &mut CatalogPageManager,
    bm: &mut BufferManager,
    index_oid: u32,
) -> Result<(), CatalogError>
```

**Validations:**
- Cannot drop an index that is referenced by a PRIMARY KEY or UNIQUE constraint.

---

### `index_lookup`

**Description:**  
Search a B-Tree index for a key. Used internally by constraint validation.

**Function:**
```rust
pub fn index_lookup(
    bm: &mut BufferManager,
    db_name: &str,
    index_name: &str,
    key_bytes: &[u8],
) -> Result<bool, CatalogError>
```

---

### `insert_index_entry`

**Description:**  
Insert a key-value pair into a B-Tree index. Handles page splits and root promotion.

**Function:**
```rust
pub fn insert_index_entry(
    bm: &mut BufferManager,
    db_name: &str,
    index_name: &str,
    key_bytes: &[u8],
    page_num: u32,
    slot_id: u32,
) -> Result<(), CatalogError>
```

---

## CatalogPageManager CRUD

The `CatalogPageManager` struct provides low-level CRUD operations on system catalog page files. All methods route I/O through the buffer manager.

### `insert_catalog_tuple`

```rust
pub fn insert_catalog_tuple(
    &mut self, bm: &mut BufferManager, catalog_name: &str, data: Vec<u8>,
) -> Result<(u32, u32), CatalogError>
```

Returns `(page_num, slot_id)` of the inserted tuple. Automatically creates a new page if the current last page lacks space.

### `read_catalog_tuple`

```rust
pub fn read_catalog_tuple(
    &self, bm: &mut BufferManager, catalog_name: &str, page_num: u32, slot_id: u32,
) -> Result<Vec<u8>, CatalogError>
```

### `update_catalog_tuple`

```rust
pub fn update_catalog_tuple(
    &mut self, bm: &mut BufferManager, catalog_name: &str,
    page_num: u32, slot_id: u32, new_data: &[u8],
) -> Result<(u32, u32), CatalogError>
```

Uses a **delete-then-reinsert** strategy to handle variable-length tuples (see [Implementation Notes](./implementation-notes.md) §3). Returns the new `(page_num, slot_id)`.

### `scan_catalog`

```rust
pub fn scan_catalog(
    &self, bm: &mut BufferManager, catalog_name: &str,
) -> Result<Vec<Vec<u8>>, CatalogError>
```

Returns all live tuples from the catalog (skips logically deleted slots with `length == 0`).

### `delete_catalog_tuple`

```rust
pub fn delete_catalog_tuple(
    &self, bm: &mut BufferManager, catalog_name: &str, page_num: u32, slot_id: u32,
) -> Result<(), CatalogError>
```

Performs a **logical delete** by zeroing the slot's length field. The space is not reclaimed immediately.

### `find_catalog_tuple`

```rust
pub fn find_catalog_tuple<F>(
    &self, bm: &mut BufferManager, catalog_name: &str, predicate: F,
) -> Result<Option<(u32, u32, Vec<u8>)>, CatalogError>
where F: Fn(&[u8]) -> bool
```

Scans the catalog and returns the first tuple matching the predicate, along with its `(page_num, slot_id)`.

> **Note:** Some APIs have undergone changes during development. See the [Implementation Notes](./implementation-notes.md) for details on deviations from the original design.
