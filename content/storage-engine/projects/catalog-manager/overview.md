---
title: Overview
sidebar_position: 1
---

# Catalog Manager — Overview

## Motivation

Prior to the Catalog Manager project, RookDB stored all metadata in a single JSON file (`database/global/catalog.json`). While simple, this approach had several limitations:

- **No scalability** — the entire catalog was serialised and deserialized on every operation, making it impractical for large numbers of tables.
- **No constraint support** — there was no mechanism to define or enforce primary keys, foreign keys, unique constraints, or NOT NULL.
- **No type metadata** — only `INT` and `TEXT` were supported, with no extensible type system.
- **No buffer integration** — catalog I/O bypassed the buffer manager entirely.
- **No object identity** — tables and columns had no unique identifiers, making references fragile.

## Goals

The Catalog Manager project addresses these limitations by introducing:

1. **Page-based catalog storage** — system catalogs are stored as slotted pages (identical format to user tables), enabling integration with the buffer manager and supporting large-scale metadata.
2. **Self-hosting architecture** — system catalog tables describe themselves, following PostgreSQL's proven design.
3. **OID system** — every database object (database, table, column, constraint, index, type) receives a persistent, unique 32-bit Object Identifier.
4. **Constraint system** — full support for PRIMARY KEY, FOREIGN KEY (with cascading actions), UNIQUE, NOT NULL, and CHECK constraints.
5. **Extended type system** — ten built-in types (INT, BIGINT, FLOAT, DOUBLE, BOOL, TEXT, VARCHAR, DATE, TIMESTAMP, BYTES) with alignment and length metadata.
6. **In-memory LRU cache** — reduces redundant page reads with automatic invalidation on every DDL operation.
7. **Dual-mode compatibility** — the system gracefully handles both the legacy JSON format and the new page-based format during migration.

## Design Principles

The design is guided by the following principles:

- **Consistency with RookDB internals** — catalog pages use the same 8 KB slotted-page layout as user tables, reusing existing page and disk infrastructure.
- **PostgreSQL conventions** — system catalog naming (`pg_database`, `pg_table`, etc.), OID-based references, and constraint semantics follow PostgreSQL precedents.
- **Separation of concerns** — the catalog module is cleanly separated into sub-modules: types, serialization, page management, constraints, indexes, cache, and OID management.
- **Write-through durability** — DDL changes are persisted immediately to the page backend; the OID counter is written to disk on every allocation to prevent reuse after a restart.

## Scope

The Catalog Manager project modifies or creates files across the following areas:

| Area | Files |
|------|-------|
| Catalog module (`src/backend/catalog/`) | `types.rs`, `catalog.rs`, `mod.rs`, `constraints.rs`, `indexes.rs`, `cache.rs`, `oid.rs`, `page_manager.rs`, `serialize.rs` |
| Buffer Manager (`src/backend/buffer_manager/`) | `buffer_manager.rs`, `mod.rs` |
| Executor (`src/backend/executor/`) | `load_csv.rs`, `seq_scan.rs` |
| Frontend (`src/frontend/`) | `menu.rs`, `database_cmd.rs`, `table_cmd.rs`, `data_cmd.rs` |
| Layout (`src/backend/`) | `layout.rs` |
| Tests (`tests/`) | `test_init_catalog.rs`, `test_load_catalog.rs`, `test_save_catalog.rs` |
