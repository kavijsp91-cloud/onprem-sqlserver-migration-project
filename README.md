# On-Prem SQL Server -> ADLS Gen2 Incremental Migration (Azure Data Factory)

Metadata-driven, watermark-based incremental pipeline that copies **20 tables** (10 dimensions + 10 facts)
from an on-premises SQL Server data warehouse into Azure Data Lake Storage Gen2 as **Parquet**.

## Architecture
```
Parent pipeline (orchestration)
 |- Lookup  : dbo.ADF_Watermark_Table (TableSchema, TableName, WatermarkColumn, LastLoadValue)
 |- ForEach : one iteration per table
     |- Execute Pipeline -> Child pipeline (incremental)
          |- Lookup : MAX(watermark column) from source table
          |- If     : LastLoadValue is NULL OR MaxValue > LastLoadValue
              |- Copy    : SQL Server -> ADLS Parquet  (full load first time, incremental afterwards)
              |- Execute Pipeline -> Stored-procedure pipeline -> dbo.UpdateWatermark
```
Source reaches Azure through a **Self-hosted Integration Runtime**.

## Output layout in ADLS
`retaildw/<TableName>/<yyyy>/<MM>/<dd>/<TableName>_<yyyyMMddHHmmss>.parquet`

## Repo layout
| Path | Content |
|---|---|
| `adf/` | ADF factory JSON (pipelines, datasets, linked services) committed by ADF Git integration |
| `sql/` | Source verification, watermark table, `UpdateWatermark` stored procedure, incremental test |
| `docs/` | Expression cheat-sheet and run screenshots |

## How to run
1. Run `sql/00..02` on the source DB (watermark table has 20 rows).
2. Create ADLS Gen2 account + container `retaildw`, ADF, and register the Self-hosted IR.
3. Create linked services/datasets, then pipelines (see `docs/adf_expressions.md`).
4. Debug the parent pipeline: run 1 = full load, run 2 = no new data (Copy skipped), then insert rows and re-run (incremental).

## Security
No keys or passwords are stored in this repository. Linked-service secrets must be re-entered when the factory is re-created.

## Teardown
All Azure resources live in one resource group; deleting the resource group removes everything and stops charges.
