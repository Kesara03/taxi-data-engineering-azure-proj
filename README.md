# Azure Medallion Data Lake

Production-grade data engineering pipeline on Azure implementing Medallion Architecture with ADLS Gen2, Azure Data Factory, and Databricks. Designed for reliable, scalable data movement.

## Architecture

```
Source Files --> ADF Pipeline --> Raw --> Autoloader --> Bronze --> PySpark --> Silver --> Gold
     |               |            |           |            |           |          |        |
  Monthly        Dynamic       ADLS       Streaming     Delta      Transform   Delta   Analytics
   CSVs         ForEach       Gen2       Ingestion     Tables     Notebooks   Tables    Ready
```

## Pipeline Layers

### Raw Layer
Landing zone for source files on ADLS Gen2.

### Bronze Layer (Ingestion)
Streaming-style ingestion using Databricks Autoloader.

**Features:**
- Schema inference enabled
- Checkpointing in lake for exactly-once processing
- Clean Delta tables as output

```python
df = spark.readStream.format("cloudFiles") \
    .option("cloudFiles.format", "csv") \
    .option("cloudFiles.schemaLocation", checkpoint_path) \
    .load(raw_path)

df.writeStream.format("delta") \
    .option("checkpointLocation", checkpoint_path) \
    .outputMode("append") \
    .toTable("bronze.table_name")
```

### Silver Layer (Transformation)
Reusable PySpark transformations orchestrated by Databricks Workflows.

**Transformation Logic:**
- Type casting
- Null value handling
- String splits for noisy fields
- Normalized time fields (date, year, month)

**Orchestration Pattern:**
```
Lookup Notebook --> Task Array --> Databricks Workflow --> Parameterized Transform Notebook
```

Single reusable notebook receives parameters from workflow iteration, keeping code minimal and behavior consistent.

### Gold Layer (Consumption)
Curated Delta tables with compact schemas ready for analytics.

**Features:**
- External tables registered over gold paths
- File preservation on table deletion
- Clear storage ownership

## Azure Data Factory Pipeline

### Dynamic Ingestion Design

**Components:**
| Component | Purpose |
|-----------|---------|
| Single parameterized dataset | Reusable across all sources |
| Configuration array | Defines source-target mappings |
| ForEach activity | Parallel monthly file copies |
| Validation step | Seed file pre-check |

**Pre-Check Logic:**
Validates required seed file exists before data movement starts, preventing silent failures and broken dependencies.

```json
{
  "activities": [
    {
      "name": "Validate Seed File",
      "type": "GetMetadata",
      "typeProperties": {
        "dataset": "SeedFileDataset"
      }
    },
    {
      "name": "ForEach Monthly File",
      "type": "ForEach",
      "dependsOn": ["Validate Seed File"],
      "typeProperties": {
        "isSequential": false,
        "items": "@pipeline().parameters.fileConfig"
      }
    }
  ]
}
```

## Security & Reliability

| Feature | Implementation |
|---------|----------------|
| Authentication | Service Principal |
| Authorization | Storage Blob Data Contributor role |
| Storage | Hierarchical Namespace enabled |
| Idempotency | Checkpoint-based processing |
| Monitoring | Pipeline run tracking |
| Lineage | Clear per-folder and per-table tracking |

## Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        ADLS Gen2                                │
├─────────────┬─────────────┬─────────────┬─────────────────────┤
│    Raw      │   Bronze    │   Silver    │        Gold         │
│             │             │             │                     │
│  Monthly    │   Delta     │   Delta     │   Delta Tables      │
│  CSV Files  │   Tables    │   Tables    │   (External)        │
│             │             │             │                     │
│  Landing    │  Autoloader │  PySpark    │   Analytics Ready   │
│  Zone       │  Ingestion  │  Transform  │   Compact Schema    │
└─────────────┴─────────────┴─────────────┴─────────────────────┘
```

## Technical Stack

| Component | Technology |
|-----------|------------|
| Storage | ADLS Gen2 |
| Orchestration | Azure Data Factory |
| Compute | Databricks |
| File Format | Delta Lake |
| Ingestion | Autoloader |
| Transform | PySpark |
| Security | Service Principal + RBAC |

## Key Design Decisions

1. **Single parameterized dataset:** Eliminates dataset proliferation, simplifies maintenance
2. **ForEach parallelism:** Maximizes throughput for monthly file ingestion
3. **Seed file validation:** Prevents cascade failures from missing dependencies
4. **Autoloader checkpointing:** Guarantees exactly-once processing
5. **Lookup + parameterized notebook:** Keeps transformation code DRY and testable
6. **External tables:** Preserves data independence from metastore

## Deployment

1. Configure Service Principal with Storage Blob Data Contributor role
2. Enable Hierarchical Namespace on ADLS Gen2
3. Deploy ADF pipeline with linked services
4. Configure Databricks workspace with ADLS mount
5. Schedule pipeline triggers

## Author

Kesara Rathnasiri
