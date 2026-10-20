# tac: Tableau as Code

[![CI](https://github.com/a-g-y/tac/actions/workflows/ci.yml/badge.svg)](https://github.com/a-g-y/tac/actions)
![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> Define Tableau data sources as clean, human-readable YAML. Validate them, review them in Git, and compile them into `.tds` files deterministically.

`tac` turns a Tableau data source (`.tds`) into a small set of readable, diffable YAML specs and back. Workbook support (`.twb`) is planned and will build on the same foundation.

---

## Why

Tableau's native XML is hostile to version control:

- **Noise**: GUIDs, ordering, and generated attributes change on every save.
- **Opacity**: calculations, relationships, and folders are buried in nested XML.
- **No review path**: a one-line formula change becomes an unreadable diff.

With `tac`, a data source is a YAML file keyed by clean slugs:

```yaml
name: snowflake_revenue_model
connection:
  type: snowflake
  database: FINANCE_DB
  schema: ANALYTICS
  server: acme.snowflakecomputing.com
  warehouse: REPORTING_WH
columns:
  customer_id:
    type: nominal
    role: dimension
    data_type: string
    label: Customer ID
calculations:
  arrratio:
    type: quantitative
    role: measure
    data_type: real
    label: ARR to MRR Ratio
    formula: "SUM([arr_amount]) / SUM([mrr_amount])"
```

Changing a formula is a one-line diff:

```diff
-    formula: "SUM([arr_amount]) / SUM([mrr_amount])"
+    formula: "SUM([arr_amount]) / NULLIF(SUM([mrr_amount]), 0)"
```

---

## Installation

```bash
pip install .            # from a checkout; installs the `tac` command
pip install ".[schema]"  # also installs pydantic for strict schema validation
```

No runtime dependencies are required. You can also run from a checkout without installing: `PYTHONPATH=src python3 -m tac --help`.

---

## Quickstart with Examples

The repository includes a ready-to-use, IP-safe example based on generic Superstore data:
- `examples/superstore.yaml`: Clean, human-authored datasource specification
- `examples/superstore.tds`: Compiled standalone Tableau Data Source XML
- `examples/superstore.tdsx`: Packaged Tableau Data Source bundle with embedded extract

### 1. Author and Compile

```bash
# Validate your YAML specification
tac validate examples/superstore.yaml

# Compile YAML directly into a standalone .tds file
tac compile examples/superstore.yaml ./output/superstore.tds
```

### 2. Decompile Existing Tableau Sources

```bash
# Decompile a .tds into clean YAML specs
tac decompile examples/superstore.tds ./specs

# Or preserve Tableau's exact internal calculation IDs (e.g. [Calculation_...])
tac decompile examples/superstore.tds ./specs --preserve-ids
```

`tac compile` runs semantic validation first and aborts on critical errors (duplicate or invalid slugs). `tac validate` reports every finding standalone, which is ideal for CI/CD gates.

---

## Working with Packaged Data Sources (`.tdsx`) and the Bundler

### Why the Bundler Matters for Extract-Based Sources

A standalone `.tds` file only stores metadata and connection parameters. When a data source connects to a local Tableau Hyper extract (`.hyper`), the `.tds` references a relative file path (e.g. `Data/Extracts/superstore.hyper`).

If the `.hyper` extract file is missing from that exact relative location, or if the data source connects to an asset that **cannot be refreshed** (such as static extracts or production databases inaccessible from local machines), Tableau Desktop cannot open the `.tds` and errors out immediately.

`tac unbundle` and `tac bundle` solve this by treating `.tdsx` archives as first-class citizens:

1. **Unbundle**: Extracts the primary `.tds` XML and packages all embedded assets (`.hyper` extracts, shapes, images) into an `assets/` directory.
2. **Decompile & Iterate**: Modify calculations, folders, descriptions, or metadata cleanly in YAML.
3. **Compile & Re-bundle**: Re-packages the generated `.tds` together with the `assets/` directory back into a `.tdsx`.

This lets you test your compiled data sources directly in Tableau Desktop offline, even when the data cannot be refreshed.

### End-to-End Walkthrough

```bash
# 1. Unbundle the packaged archive into XML and assets
tac unbundle examples/superstore.tdsx ./extracted_superstore

# 2. Decompile the extracted TDS into human-readable YAML
tac decompile ./extracted_superstore/original.tds ./specs

# 3. Edit ./specs/datasource.yaml (e.g. update formulas or add KPIs), then validate
tac validate ./specs

# 4. Compile the updated YAML spec into a new TDS
tac compile ./specs ./output/superstore.tds

# 5. Re-bundle into a self-contained TDSX with the original .hyper extract asset
tac bundle ./output/superstore.tds ./output/superstore.tdsx --assets ./extracted_superstore/assets
```

The resulting `output/superstore.tdsx` can be opened immediately in Tableau Desktop with all data intact.

---

### Commands

| Command | Description |
| :--- | :--- |
| `tac decompile <source> <output_dir> [--preserve-ids]` | Decompile a `.tds` into modular YAML specs. Use `--preserve-ids` to retain internal XML calculation IDs. |
| `tac compile <source> <output_file>` | Compile modular YAML specs (or a single `.yaml` file) into a `.tds`. |
| `tac validate <path>` | Validate a spec file or directory (semantic rules, plus strict schema checks when `pydantic` is installed). |
| `tac unbundle <archive> <output_dir>` | Extract a packaged archive (`.tdsx`, `.twbx`) into XML and `assets/`. |
| `tac bundle <xml_file> <output_archive> [--assets DIR]` | Package a `.tds` and assets into a `.tdsx` container for testing. |
| `tac clean <source> [-o OUT]` | Strip noise (thumbnails, window state) from a `.twb`. |
| `tac diff <dir_a> <dir_b>` | Compare two decompiled spec directories. |

---

## Validation

`tac validate` checks, among other things:

- Unique, well-formed calculation slugs and display names
- No slug collisions between columns and calculations
- Balanced brackets and parentheses in formulas
- Hierarchies, sets, and groups referencing declared fields only
- At most one filter per field

`compile` runs these checks automatically and stops on critical errors.

---

## Documentation

- **[Data Source Schema Reference](docs/schema_reference.md)**: Syntax reference for database connections, tables, columns, calculations, and extracts.
- **[Architecture & Design](docs/architecture.md)**: Deep dive into the compiler, normalizer, and AST pipeline.

---

## Python API

```python
from tac import DatasourceDecompiler, DatasourceCompiler, validate_spec_path
from tac.schema import DataSource  # typed models for the datasource spec

# Decompile (optionally pass preserve_ids=True)
DatasourceDecompiler.from_file("my_datasource.tds", preserve_ids=True).decompile("specs")

# Validate (returns {file: [errors]})
results = validate_spec_path("specs")

# Compile
xml = DatasourceCompiler("specs").compile()
with open("my_datasource.tds", "w", encoding="utf-8") as f:
    f.write(xml)
```

`tac.datasources` re-exports the datasource-focused API in one place.

---

## Status

- **v0.1**: data sources (`.tds` / `.tdsx`).
- **Planned**: workbooks, worksheets, and dashboards, built on the same schema package (`tac.schema`).

## Requirements

- Python 3.10+
- Optional: `pydantic` v2 for strict schema validation in `tac validate` (`pip install ".[schema]"`)
