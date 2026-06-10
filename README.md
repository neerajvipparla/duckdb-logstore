<div align="center">
  <picture>
    <source media="(prefers-color-scheme: light)" srcset="logo/DuckDB_Logo-horizontal.svg">
    <source media="(prefers-color-scheme: dark)" srcset="logo/DuckDB_Logo-horizontal-dark-mode.svg">
    <img alt="DuckDB logo" src="logo/DuckDB_Logo-horizontal.svg" height="100">
  </picture>
</div>
<br>

<p align="center">
  <a href="https://github.com/duckdb/duckdb/actions"><img src="https://github.com/duckdb/duckdb/actions/workflows/Main.yml/badge.svg?branch=main" alt="Github Actions Badge"></a>
  <a href="https://discord.gg/tcvwpjfnZx"><img src="https://shields.io/discord/909674491309850675" alt="discord" /></a>
  <a href="https://github.com/duckdb/duckdb/releases/"><img src="https://img.shields.io/github/v/release/duckdb/duckdb?color=brightgreen&display_name=tag&logo=duckdb&logoColor=white" alt="Latest Release"></a>
</p>

## DuckDB

DuckDB is a high-performance analytical database system. It is designed to be fast, reliable, portable, and easy to use. DuckDB provides a rich SQL dialect with support far beyond basic SQL. DuckDB supports arbitrary and nested correlated subqueries, window functions, collations, complex types (arrays, structs, maps), and [several extensions designed to make SQL easier to use](https://duckdb.org/docs/current/sql/dialect/friendly_sql.html).

DuckDB is available as a [standalone CLI application](https://duckdb.org/docs/current/clients/cli/overview) and has clients for [Python](https://duckdb.org/docs/current/clients/python/overview), [R](https://duckdb.org/docs/current/clients/r), [Java](https://duckdb.org/docs/current/clients/java), [Wasm](https://duckdb.org/docs/current/clients/wasm/overview), etc., with deep integrations with packages such as [pandas](https://duckdb.org/docs/guides/python/sql_on_pandas) and [dplyr](https://duckdb.org/docs/current/clients/r#duckplyr-dplyr-api).

For more information on using DuckDB, please refer to the [DuckDB documentation](https://duckdb.org/docs/current/).

## Installation

If you want to install DuckDB, please see [our installation page](https://duckdb.org/docs/installation/) for instructions.

## Data Import

For CSV files and Parquet files, data import is as simple as referencing the file in the FROM clause:

```sql
SELECT * FROM 'myfile.csv';
SELECT * FROM 'myfile.parquet';
```

Refer to our [Data Import](https://duckdb.org/docs/current/data/overview) section for more information.

## SQL Reference

The documentation contains a [SQL introduction and reference](https://duckdb.org/docs/current/sql/introduction).

## Development

For development, DuckDB requires [CMake](https://cmake.org), Python 3 and a `C++17` compliant compiler. In the root directory, run `make` to compile the sources. For development, use `make debug` to build a non-optimized debug version. You should run `make unit` and `make allunit` to verify that your version works properly after making changes. To test performance, you can run `BUILD_BENCHMARK=1 BUILD_TPCH=1 make` and then perform several standard benchmarks from the root directory by executing `./build/release/benchmark/benchmark_runner`. The details of benchmarks are in our [Benchmark Guide](benchmark/README.md).

Please also refer to our [Build Guide](https://duckdb.org/docs/current/dev/building/overview) and [Contribution Guide](CONTRIBUTING.md).

## Support

See the [Support Options](https://ducklabs.com/support/) page and the dedicated [`endoflife.date`](https://endoflife.date/duckdb) page.

## Append-Only Mode

This fork patches DuckDB to enforce **append-only semantics** at the binder level. Once a row is inserted, it cannot be mutated or removed — enforced inside the engine itself, not at the application layer.

### What is blocked

| Operation | Error |
|---|---|
| `UPDATE` | `Append-only mode: UPDATE is not permitted. This store is immutable after insert.` |
| `DELETE` | `Append-only mode: DELETE is not permitted. This store is immutable after insert.` |
| `MERGE INTO` | `Append-only mode: MERGE INTO is not permitted. This store is immutable after insert.` |
| `ALTER TABLE ... DROP COLUMN` | `Append-only mode: REMOVE_COLUMN is not permitted.` |
| `ALTER TABLE ... ALTER COLUMN TYPE` | `Append-only mode: MODIFY_COLUMN is not permitted.` |

### What is still allowed

`INSERT`, `SELECT`, `CREATE TABLE`, `DROP TABLE`, `ALTER TABLE ... ADD COLUMN`, `ALTER TABLE ... RENAME COLUMN/TABLE` — all work normally.

### How it works

All five guards live in a single function: `Binder::Bind()` in `src/planner/binder.cpp`. The binder is the first stage that has full type and statement information, so the error is raised before any execution plan is produced — no storage or optimizer code is touched.

### Using with Go

The Go package (`github.com/marcboeker/go-duckdb`) compiles DuckDB from its amalgamation source via CGO. To use this append-only build:

```bash
# 1. Generate the amalgamation from this patched source
make amalgamation

# 2. Vendor the Go package and replace its amalgamation
go mod vendor
cp build/amalgamation/duckdb.cpp vendor/github.com/marcboeker/go-duckdb/duckdb.cpp
cp build/amalgamation/duckdb.hpp vendor/github.com/marcboeker/go-duckdb/duckdb.hpp

# 3. Build — the append-only engine is now statically linked into your binary
go build ./...
```

No runtime download. The restrictions are baked into the compiled binary.
