# AWS-Glue
## High-level flow

![Image](https://www.tothenew.com/blog/wp-ttn-blog/uploads/2023/10/Architecture-1024x684.png)

![Image](https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2021/02/12/BDB-978-P-1-3WHITE.png)

![Image](https://docs.aws.amazon.com/images/glue/latest/dg/images/HowItWorks-overview.png)

1. **Amazon RDS snapshot** → exported to **Amazon S3**
2. Snapshot export produces **columnar files (usually Parquet)**
3. **AWS Glue Crawler** scans the S3 location
4. **AWS Glue Data Catalog** tables are created automatically

> Now let’s open the black box

## What *exactly* is inside the S3 export from RDS?

When you export an RDS snapshot:

### Export format (important)

* Data is written as:

  * **Apache Parquet** (most common)
  * OR **ORC** (depending on config)
* Folder structure typically looks like:

  ```
  s3://rds-export-bucket/
    database_name/
      schema_name/
        table_name/
          part-00001.snappy.parquet
          part-00002.snappy.parquet
  ```

### Why Parquet?

Because Parquet files contain:

* **Embedded schema metadata**
* Column names
* Data types
* Nullable flags

> This is the **key reason Glue does NOT need RDS credentials** to infer schema.

## What does a Glue Crawler *really* do?

![Image](https://docs.aws.amazon.com/images/glue/latest/dg/images/PopulateCatalog-overview.png)

![Image](https://cn-northwest-1.prod.ces.corpus.marketing.aws.a2z.org.cn/rebrand-images/d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2023/08/20/BDB-2626_image001.png)

![Image](https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2021/12/21/BDB-1662-image002.png)

A Glue crawler is basically:

> **A distributed metadata scanner running on Spark**

### Internally, the crawler does this:

### Step 1: Path discovery

* Crawls every folder under the S3 prefix
* Detects **dataset boundaries** using:

  * Folder depth
  * File naming patterns
  * Compression types

Example:

```
/sales_db/public/orders/  → one dataset
/sales_db/public/customers/ → another dataset
```

---

### Step 2: File-type detection

For each dataset:

* Detects format:

  * Parquet
  * ORC
  * JSON
  * CSV
* Chooses **built-in classifier**

> For RDS exports, it almost always selects:

> **Parquet classifier**

### Step 3: Schema extraction (this is the magic)

Glue **does NOT read full data**.

Instead it:

* Opens **Parquet file footers**
* Reads:

  * Column names
  * Data types (int, bigint, decimal, timestamp, string)
  * Nested structures (if any)

Example extracted schema:

```text
order_id     bigint
order_date   timestamp
amount       decimal(10,2)
customer_id  bigint
```

No JDBC. No SQL. No RDS access.

### Step 4: Schema merging

If multiple files exist:

* Glue samples multiple Parquet files
* Merges schemas
* Resolves:

  * Optional vs required fields
  * Type widening (int → bigint)

> If schemas conflict → crawler logs warnings

## What is the **Glue Connection** used for then?

This is a **very common confusion**.

### NOT used in your case

For **S3-based crawlers**, Glue **does not use the connection at all** for schema discovery.

### Connections are only used when:

* Crawling **JDBC sources**

  * RDS directly
  * Aurora
  * Redshift
* Crawling **private VPC resources**

| Source type  | Uses Glue Connection? |
| ------------ | --------------------- |
| S3 Parquet   |  No                  |
| S3 CSV/JSON  |  No                  |
| RDS via JDBC |  Yes                 |
| On-prem DB   |  Yes                 |

> Your crawler works **purely via IAM + S3 access**.

## How does Glue decide databases & table names?

Glue applies **naming rules** based on folder structure.

Example S3 path:

```
s3://bucket/export/mydb/public/orders/
```

Glue generates:

* **Database**: `mydb`
* **Table**: `orders`
* **Columns**: extracted from Parquet

You can control this using:

* Crawler “Table level”
* Database prefix
* Table prefix
* Partition inference settings

## What exactly gets stored in Glue Data Catalog?

![Image](https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b8/2021/09/28/BDB-1608-image003.png)

![Image](https://website-assets.atlan.com/img/AWS_glue_environment.webp)

![Image](https://docs.aws.amazon.com/images/glue/latest/dg/images/catalog-table-version-comparison.png)

For each table, Glue stores:

### Metadata only (NO DATA)

* Column names & data types
* Table location (S3 path)
* Input/output formats
* SerDe library
* Partition keys (if detected)

Equivalent to Hive Metastore.

This is why services like:

* Athena
* Redshift Spectrum
* EMR
  can query it instantly.

## Why this works so well for RDS snapshot exports

Because RDS snapshot export:

* Preserves **exact DB schema**
* Writes **columnar metadata-rich files**
* Uses **stable folder hierarchy**

Glue + Parquet = perfect match

## End-to-end internal flow (one-line)

> RDS snapshot → Parquet files in S3 → Glue crawler reads Parquet metadata → builds schema → registers tables in Glue Data Catalog → queryable by Athena/EMR

## Where failures usually happen (important for you)

Since you’re doing CDC / pipelines, watch for:

* ❌ KMS permissions on S3 objects
* ❌ Schema evolution conflicts
* ❌ Mixed file formats in same folder
* ❌ Multiple DBs dumped into one prefix

---
