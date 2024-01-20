<!-- markdownlint-disable-next-line MD041 -->
<p align="center">
    <img src="https://raw.githubusercontent.com/dbt-athena/dbt-athena/main/static/images/dbt-athena-long.png" />
    <a href="https://pypi.org/project/dbt-athena-community/">
      <img src="https://badge.fury.io/py/dbt-athena-community.svg" />
    </a>
    <a target="_blank" href="https://pypi.org/project/dbt-athena-community/" style="background:none">
      <img src="https://img.shields.io/pypi/pyversions/dbt-athena-community">
    </a>
    <a href="https://pycqa.github.io/isort/">
      <img src="https://img.shields.io/badge/%20imports-isort-%231674b1?style=flat&labelColor=ef8336" />
    </a>
    <a href="https://github.com/psf/black"><img src="https://img.shields.io/badge/code%20style-black-000000.svg" /></a>
    <a href="https://github.com/python/mypy"><img src="https://www.mypy-lang.org/static/mypy_badge.svg" /></a>
    <a href="https://pepy.tech/project/dbt-athena-community">
      <img src="https://static.pepy.tech/badge/dbt-athena-community/month" />
    </a>
</p>
<!-- TOC -->

- [Features](#features)
  - [Quick Start](#quick-start)
    - [Installation](#installation)
    - [Prerequisites](#prerequisites)
    - [Credentials](#credentials)
    - [Configuring your profile](#configuring-your-profile)
    - [Additional information](#additional-information)
  - [Models](#models)
    - [Table Configuration](#table-configuration)
    - [Table location](#table-location)
    - [Incremental models](#incremental-models)
    - [On schema change](#on-schema-change)
    - [Iceberg](#iceberg)
    - [Highly available table (HA)](#highly-available-table-ha)
      - [HA Known issues](#ha-known-issues)
  - [Snapshots](#snapshots)
    - [Timestamp strategy](#timestamp-strategy)
    - [Check strategy](#check-strategy)
    - [Hard-deletes](#hard-deletes)
    - [AWS Lakeformation integration](#aws-lakeformation-integration)
    - [Working example](#working-example)
    - [Snapshots Known issues](#snapshots-known-issues)
  - [Contracts](#contracts)
  - [Contributing](#contributing)
  - [Contributors ✨](#contributors-)
<!-- TOC -->

# Features

- Supports dbt version `1.7.*`
- Support for Python
- Supports [seeds][seeds]
- Correctly detects views and their columns
- Supports [table materialization][table]
  - [Iceberg tables][athena-iceberg] are supported **only with Athena Engine v3** and **a unique table location**
    (see table location section below)
  - Hive tables are supported by both Athena engines.
- Supports [incremental models][incremental]
  - On Iceberg tables :
    - Supports the use of `unique_key` only with the `merge` strategy
    - Supports the `append` strategy
  - On Hive tables :
    - Supports two incremental update strategies: `insert_overwrite` and `append`
    - Does **not** support the use of `unique_key`
- Supports [snapshots][snapshots]
- Supports [Python models][python-models]

[seeds]: https://docs.getdbt.com/docs/building-a-dbt-project/seeds

[incremental]: https://docs.getdbt.com/docs/build/incremental-models

[table]: https://docs.getdbt.com/docs/build/materializations#table

[python-models]: https://docs.getdbt.com/docs/build/python-models#configuring-python-models

[athena-iceberg]: https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html

[snapshots]: https://docs.getdbt.com/docs/build/snapshots

## Quick Start

### Installation

- `pip install dbt-athena-community`
- Or `pip install git+https://github.com/dbt-athena/dbt-athena.git`

### Prerequisites

To start, you will need an S3 bucket, for instance `my-bucket` and an Athena database:

```sql
CREATE DATABASE IF NOT EXISTS analytics_dev
COMMENT 'Analytics models generated by dbt (development)'
LOCATION 's3://my-bucket/'
WITH DBPROPERTIES ('creator'='Foo Bar', 'email'='foo@bar.com');
```

Notes:

- Take note of your AWS region code (e.g. `us-west-2` or `eu-west-2`, etc.).
- You can also use [AWS Glue](https://docs.aws.amazon.com/athena/latest/ug/glue-athena.html) to create and manage Athena
  databases.

### Credentials

Credentials can be passed directly to the adapter, or they can
be [determined automatically](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html) based
on `aws cli`/`boto3` conventions.
You can either:

- configure `aws_access_key_id` and `aws_secret_access_key`
- configure `aws_profile_name` to match a profile defined in your AWS credentials file
  Checkout dbt profile configuration below for details.

### Configuring your profile

A dbt profile can be configured to run against AWS Athena using the following configuration:

| Option                | Description                                                                              | Required? | Example                                    |
| --------------------- | ---------------------------------------------------------------------------------------- | --------- | ------------------------------------------ |
| s3_staging_dir        | S3 location to store Athena query results and metadata                                   | Required  | `s3://bucket/dbt/`                         |
| s3_data_dir           | Prefix for storing tables, if different from the connection's `s3_staging_dir`           | Optional  | `s3://bucket2/dbt/`                        |
| s3_data_naming        | How to generate table paths in `s3_data_dir`                                             | Optional  | `schema_table_unique`                      |
| s3_tmp_table_dir      | Prefix for storing temporary tables, if different from the connection's `s3_data_dir`    | Optional  | `s3://bucket3/dbt/`                        |
| region_name           | AWS region of your Athena instance                                                       | Required  | `eu-west-1`                                |
| schema                | Specify the schema (Athena database) to build models into (lowercase **only**)           | Required  | `dbt`                                      |
| database              | Specify the database (Data catalog) to build models into (lowercase **only**)            | Required  | `awsdatacatalog`                           |
| poll_interval         | Interval in seconds to use for polling the status of query results in Athena             | Optional  | `5`                                        |
| debug_query_state     | Flag if debug message with Athena query state is needed                                  | Optional  | `false`                                    |
| aws_access_key_id     | Access key ID of the user performing requests.                                           | Optional  | `AKIAIOSFODNN7EXAMPLE`                     |
| aws_secret_access_key | Secret access key of the user performing requests                                        | Optional  | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| aws_profile_name      | Profile to use from your AWS shared credentials file.                                    | Optional  | `my-profile`                               |
| work_group            | Identifier of Athena workgroup                                                           | Optional  | `my-custom-workgroup`                      |
| num_retries           | Number of times to retry a failing query                                                 | Optional  | `3`                                        |
| spark_work_group      | Identifier of Athena Spark workgroup                                           | Optional  | `my-spark-workgroup`                       |
| num_boto3_retries     | Number of times to retry boto3 requests (e.g. deleting S3 files for materialized tables) | Optional  | `5`                                        |
| seed_s3_upload_args   | Dictionary containing boto3 ExtraArgs when uploading to S3                               | Optional  | `{"ACL": "bucket-owner-full-control"}`     |
| lf_tags_database      | Default LF tags for new database if it's created by dbt                                  | Optional  | `tag_key: tag_value`                       |

**Example profiles.yml entry:**

```yaml
athena:
  target: dev
  outputs:
    dev:
      type: athena
      s3_staging_dir: s3://athena-query-results/dbt/
      s3_data_dir: s3://your_s3_bucket/dbt/
      s3_data_naming: schema_table
      s3_tmp_table_dir: s3://your_s3_bucket/temp/
      region_name: eu-west-1
      schema: dbt
      database: awsdatacatalog
      threads: 4
      aws_profile_name: my-profile
      work_group: my-workgroup
      spark_work_group: my-spark-workgroup
      seed_s3_upload_args:
        ACL: bucket-owner-full-control
```

### Additional information

- `threads` is supported
- `database` and `catalog` can be used interchangeably

## Models

### Table Configuration

- `external_location` (`default=none`)
  - If set, the full S3 path in which the table will be saved.
  - It works only with incremental models.
  - Does not work with Hive table with `ha` set to true.
- `partitioned_by` (`default=none`)
  - An array list of columns by which the table will be partitioned
  - Limited to creation of 100 partitions (*currently*)
- `bucketed_by` (`default=none`)
  - An array list of columns to bucket data, ignored if using Iceberg
- `bucket_count` (`default=none`)
  - The number of buckets for bucketing your data, ignored if using Iceberg
- `table_type` (`default='hive'`)
  - The type of table
  - Supports `hive` or `iceberg`
- `ha` (`default=false`)
  - If the table should be built using the high-availability method. This option is only available for Hive tables
    since it is by default for Iceberg tables (see the section [below](#highly-available-table-ha))
- `format` (`default='parquet'`)
  - The data format for the table
  - Supports `ORC`, `PARQUET`, `AVRO`, `JSON`, `TEXTFILE`
- `write_compression` (`default=none`)
  - The compression type to use for any storage format that allows compression to be specified. To see which options are
    available, check out [CREATE TABLE AS][create-table-as]
- `field_delimiter` (`default=none`)
  - Custom field delimiter, for when format is set to `TEXTFILE`
- `table_properties`: table properties to add to the table, valid for Iceberg only
- `native_drop`: Relation drop operations will be performed with SQL, not direct Glue API calls. No S3 calls will be
  made to manage data in S3. Data in S3 will only be cleared up for Iceberg
  tables [see AWS docs](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-managing-tables.html). Note that
  Iceberg DROP TABLE operations may timeout if they take longer than 60 seconds.
- `seed_by_insert` (`default=false`)
  - default behaviour uploads seed data to S3. This flag will create seeds using an SQL insert statement
  - large seed files cannot use `seed_by_insert`, as the SQL insert statement would
    exceed [the Athena limit of 262144 bytes](https://docs.aws.amazon.com/athena/latest/ug/service-limits.html)
- `force_batch` (`default=false`)
  - Skip creating the table as ctas and run the operation directly in batch insert mode.
  - This is particularly useful when the standard table creation process fails due to partition limitations,
  allowing you to work with temporary tables and persist the dataset more efficiently.
- `lf_tags_config` (`default=none`)
  - [AWS lakeformation](#aws-lakeformation-integration) tags to associate with the table and columns
  - `enabled` (`default=False`) whether LF tags management is enabled for a model
  - `tags` dictionary with tags and their values to assign for the model
  - `tags_columns` dictionary with a tag key, value and list of columns they must be assigned to
  - `lf_inherited_tags` (`default=none`)
    - List of Lake Formation tag keys that are intended to be inherited from the database level and thus shouldn't be
      removed during association of those defined in `lf_tags_config`
      - i.e., the default behavior of `lf_tags_config` is to be exhaustive and first remove any pre-existing tags from
        tables and columns before associating the ones currently defined for a given model
      - This breaks tag inheritance as inherited tags appear on tables and columns like those associated directly

```sql
{{
  config(
    materialized='incremental',
    incremental_strategy='append',
    on_schema_change='append_new_columns',
    table_type='iceberg',
    schema='test_schema',
    lf_tags_config={
          'enabled': true,
          'tags': {
            'tag1': 'value1',
            'tag2': 'value2'
          },
          'tags_columns': {
            'tag1': {
              'value1': ['column1', 'column2'],
              'value2': ['column3', 'column4']
            }
          },
          'inherited_tags': ['tag1', 'tag2']
    }
  )
}}
```

- format for `dbt_project.yml`:

```yaml
  +lf_tags_config:
    enabled: true
    tags:
      tag1: value1
      tag2: value2
    tags_columns:
      tag1:
        value1: [ column1, column2 ]
    inherited_tags: [ tag1, tag2 ]
```

- `lf_grants` (`default=none`)
  - lakeformation grants config for data_cell filters
  - format:

  ```python
  lf_grants={
          'data_cell_filters': {
              'enabled': True | False,
              'filters': {
                  'filter_name': {
                      'row_filter': '<filter_condition>',
                      'principals': ['principal_arn1', 'principal_arn2']
                  }
              }
          }
      }
  ```

> Notes:  
>
> - `lf_tags` and `lf_tags_columns` configs support only attaching lf tags to corresponding resources.
> We recommend managing LF Tags permissions somewhere outside dbt. For example, you may use
> [terraform](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lakeformation_permissions) or
> [aws cdk](https://docs.aws.amazon.com/cdk/api/v1/docs/aws-lakeformation-readme.html) for such purpose.
> - `data_cell_filters` management can't be automated outside dbt because the filter can't be attached to the table
> which doesn't exist. Once you `enable` this config, dbt will set all filters and their permissions during every
> dbt run. Such approach keeps the actual state of row level security configuration actual after every dbt run and
> apply changes if they occur: drop, create, update filters and their permissions.
> - Any tags listed in `lf_inherited_tags` should be strictly inherited from the database level and never overridden at
    the table and column level
>   - Currently `dbt-athena` does not differentiate between an inherited tag association and an override of same it made
>     previously
>   - e.g. If an inherited tag is overridden by an `lf_tags_config` value in one DBT run, and that override is removed
      prior to a subsequent run, the prior override will linger and no longer be encoded anywhere (in e.g. Terraform
      where the inherited value is configured nor in the DBT project where the override previously existed but now is
      gone)

[create-table-as]: https://docs.aws.amazon.com/athena/latest/ug/create-table-as.html#ctas-table-properties

### Table location

The location in which a table is saved is determined by:

1. If `external_location` is defined, that value is used.
2. If `s3_data_dir` is defined, the path is determined by that and `s3_data_naming`
3. If `s3_data_dir` is not defined, data is stored under `s3_staging_dir/tables/`

Here all the options available for `s3_data_naming`:

- `unique`: `{s3_data_dir}/{uuid4()}/`
- `table`: `{s3_data_dir}/{table}/`
- `table_unique`: `{s3_data_dir}/{table}/{uuid4()}/`
- `schema_table`: `{s3_data_dir}/{schema}/{table}/`
- `s3_data_naming=schema_table_unique`: `{s3_data_dir}/{schema}/{table}/{uuid4()}/`

It's possible to set the `s3_data_naming` globally in the target profile, or overwrite the value in the table config,
or setting up the value for groups of model in dbt_project.yml.

> Note: when using a workgroup with a default output location configured, `s3_data_naming` and any configured buckets
> are ignored and the location configured in the workgroup is used.

### Incremental models

Support for [incremental models](https://docs.getdbt.com/docs/build/incremental-models).

These strategies are supported:

- `insert_overwrite` (default): The insert overwrite strategy deletes the overlapping partitions from the destination
  table, and then inserts the new records from the source. This strategy depends on the `partitioned_by` keyword! If no
  partitions are defined, dbt will fall back to the `append` strategy.
- `append`: Insert new records without updating, deleting or overwriting any existing data. There might be duplicate
  data (e.g. great for log or historical data).
- `merge`: Conditionally updates, deletes, or inserts rows into an Iceberg table. Used in combination with `unique_key`.
  Only available when using Iceberg.

### On schema change

`on_schema_change` is an option to reflect changes of schema in incremental models.
The following options are supported:

- `ignore` (default)
- `fail`
- `append_new_columns`
- `sync_all_columns`

For details, please refer
to [dbt docs](https://docs.getdbt.com/docs/build/incremental-models#what-if-the-columns-of-my-incremental-model-change).

### Iceberg

The adapter supports table materialization for Iceberg.

To get started just add this as your model:

```sql
{{ config(
    materialized='table',
    table_type='iceberg',
    format='parquet',
    partitioned_by=['bucket(user_id, 5)'],
    table_properties={
     'optimize_rewrite_delete_file_threshold': '2'
     }
) }}

select 'A'          as user_id,
       'pi'         as name,
       'active'     as status,
       17.89        as cost,
       1            as quantity,
       100000000    as quantity_big,
       current_date as my_date
```

Iceberg supports bucketing as hidden partitions, therefore use the `partitioned_by` config to add specific bucketing
conditions.

Iceberg supports several table formats for data : `PARQUET`, `AVRO` and `ORC`.

It is possible to use Iceberg in an incremental fashion, specifically two strategies are supported:

- `append`: New records are appended to the table, this can lead to duplicates.
- `merge`: Performs an upsert (and optional delete), where new records are added and existing records are updated. Only
  available with Athena engine version 3.
  - `unique_key` **(required)**: columns that define a unique record in the source and target tables.
  - `incremental_predicates` (optional): SQL conditions that enable custom join clauses in the merge statement. This can
    be useful for improving performance via predicate pushdown on the target table.
  - `delete_condition` (optional): SQL condition used to identify records that should be deleted.
  - `update_condition` (optional): SQL condition used to identify records that should be updated.
  - `insert_condition` (optional): SQL condition used to identify records that should be inserted.
    - `incremental_predicates`, `delete_condition`, `update_condition` and `insert_condition` can include any column of
      the incremental table (`src`) or the final table (`target`).
      Column names must be prefixed by either `src` or `target` to prevent a `Column is ambiguous` error.

`delete_condition` example:

```sql
{{ config(
    materialized='incremental',
    table_type='iceberg',
    incremental_strategy='merge',
    unique_key='user_id',
    incremental_predicates=["src.quantity > 1", "target.my_date >= now() - interval '4' year"],
    delete_condition="src.status != 'active' and target.my_date < now() - interval '2' year",
    format='parquet'
) }}

select 'A' as user_id,
       'pi' as name,
       'active' as status,
       17.89 as cost,
       1 as quantity,
       100000000 as quantity_big,
       current_date as my_date
```

`update_condition` example:

```sql
{{ config(
        materialized='incremental',
        incremental_strategy='merge',
        unique_key=['id'],
        update_condition='target.id > 1',
        schema='sandbox'
    )
}}

{% if is_incremental() %}

select * from (
    values
    (1, 'v1-updated')
    , (2, 'v2-updated')
) as t (id, value)

{% else %}

select * from (
    values
    (-1, 'v-1')
    , (0, 'v0')
    , (1, 'v1')
    , (2, 'v2')
) as t (id, value)

{% endif %}
```

`insert_condition` example:

```sql
{{ config(
        materialized='incremental',
        incremental_strategy='merge',
        unique_key=['id'],
        insert_condition='target.status != 0',
        schema='sandbox'
    )
}}

select * from (
    values
    (1, 0)
    , (2, 1)
) as t (id, status)

```

### Highly available table (HA)

The current implementation of the table materialization can lead to downtime, as target table is dropped and re-created.
To have the less destructive behavior it's possible to use the `ha` config on your `table` materialized models.
It leverages the table versions feature of glue catalog, creating a tmp table and swapping the target table to the
location of the tmp table. This materialization is only available for `table_type=hive` and requires using unique
locations. For iceberg, high availability is by default.

```sql
{{ config(
    materialized='table',
    ha=true,
    format='parquet',
    table_type='hive',
    partitioned_by=['status'],
    s3_data_naming='table_unique'
) }}

select 'a'      as user_id,
       'pi'     as user_name,
       'active' as status
union all
select 'b'        as user_id,
       'sh'       as user_name,
       'disabled' as status
```

By default, the materialization keeps the last 4 table versions, you can change it by setting `versions_to_keep`.

#### HA Known issues

- When swapping from a table with partitions to a table without (and the other way around), there could be a little
  downtime.
  In case high performances are needed consider bucketing instead of partitions
- By default, Glue "duplicates" the versions internally, so the last two versions of a table point to the same location
- It's recommended to have `versions_to_keep` >= 4, as this will avoid having the older location removed
- The macro `athena__end_of_time` needs to be overwritten by the user if using Athena engine v3 since it requires a
  precision parameter for timestamps

## Snapshots

The adapter supports snapshot materialization. It supports both timestamp and check strategy. To create a snapshot
create a snapshot file in the snapshots directory. If the directory does not exist create one.

### Timestamp strategy

To use the timestamp strategy refer to
the [dbt docs](https://docs.getdbt.com/docs/build/snapshots#timestamp-strategy-recommended)

### Check strategy

To use the check strategy refer to the [dbt docs](https://docs.getdbt.com/docs/build/snapshots#check-strategy)

### Hard-deletes

The materialization also supports invalidating hard deletes. Check
the [docs](https://docs.getdbt.com/docs/build/snapshots#hard-deletes-opt-in) to understand usage.

### AWS Lakeformation integration

The adapter implements AWS Lakeformation tags management in the following way:

- you can enable or disable lf-tags management via [config](#table-configuration) (disabled by default)
- once you enable the feature, lf-tags will be updated on every dbt run
- first, all lf-tags for columns are removed to avoid inheritance issues
- then all redundant lf-tags are removed from table and actual tags from config are applied
- finally, lf-tags for columns are applied

It's important to understand the following points:

- dbt does not manage lf-tags for database
- dbt does not manage lakeformation permissions

That's why you should handle this by yourself manually or using some automation tools like terraform, AWS CDK etc.  
You may find the following links useful to manage that:

<!-- markdownlint-disable -->
* [terraform aws_lakeformation_permissions](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lakeformation_permissions)
* [terraform aws_lakeformation_resource_lf_tags](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lakeformation_resource_lf_tags)
<!-- markdownlint-restore -->

## Python Models

The adapter supports python models using [`spark`](https://docs.aws.amazon.com/athena/latest/ug/notebooks-spark.html).

### Setup

- A spark enabled work group created in athena
- Spark execution role granted access to Athena, Glue and S3
- The spark work group is added to the ~/.dbt/profiles.yml file and the profile is referenced in dbt_project.yml
  that will be created. It is recommended to keep this same as threads.

### Spark specific table configuration

- `timeout` (`default=43200`)
  - Time out in seconds for each python model execution. Defaults to 12 hours/43200 seconds.
- `spark_encryption` (`default=false`)
  - If this flag is set to true, encrypts data in transit between Spark nodes and also encrypts data at rest stored
   locally by Spark.
- `spark_cross_account_catalog` (`default=false`)
  - In spark, you can query the external account catalog and for that the consumer account has to be configured to
   access the producer catalog.
  - If this flag is set to true, "/" can be used as the glue catalog separator.
   Ex: 999999999999/mydatabase.cloudfront_logs (*where *999999999999* is the external catalog id*)
- `spark_requester_pays` (`default=false`)
  - When an Amazon S3 bucket is configured as requester pays, the account of the user running the query is charged for
   data access and data transfer fees associated with the query.
  - If this flag is set to true, requester pays S3 buckets are enabled in Athena for Spark.

### Spark notes

- A session is created for each unique engine configuration defined in the models that are part of the invocation.
- A session's idle timeout is set to 10 minutes. Within the timeout period, if there is a new calculation
 (spark python model) ready for execution and the engine configuration matches, the process will reuse the same session.
- Number of python models running at a time depends on the `threads`.  Number of sessions created for the entire run
 depends on number of unique engine configurations and availability of session to maintain threads concurrency.
- For iceberg table, it is recommended to use table_properties configuration to set the format_version to 2. This is to
 maintain compatability between iceberg tables created by Trino with those created by Spark.

### Example models

#### Simple pandas model

```python
import pandas as pd


def model(dbt, session):
    dbt.config(materialized="table")

    model_df = pd.DataFrame({"A": [1, 2, 3, 4]})

    return model_df
```

#### Simple spark

```python
def model(dbt, spark_session):
    dbt.config(materialized="table")

    data = [(1,), (2,), (3,), (4,)]

    df = spark_session.createDataFrame(data, ["A"])

    return df
```

#### Spark incremental

```python
def model(dbt, spark_session):
    dbt.config(materialized="incremental")
    df = dbt.ref("model")

    if dbt.is_incremental:
        max_from_this = (
            f"select max(run_date) from {dbt.this.schema}.{dbt.this.identifier}"
        )
        df = df.filter(df.run_date >= spark_session.sql(max_from_this).collect()[0][0])

    return df
```

#### Config spark model

```python
def model(dbt, spark_session):
    dbt.config(
        materialized="table",
        engine_config={
            "CoordinatorDpuSize": 1,
            "MaxConcurrentDpus": 3,
            "DefaultExecutorDpuSize": 1
        },
        spark_encryption=True,
        spark_cross_account_catalog=True,
        spark_requester_pays=True
        polling_interval=15,
        timeout=120,
    )

    data = [(1,), (2,), (3,), (4,)]

    df = spark_session.createDataFrame(data, ["A"])

    return df
```

#### Create pySpark udf using imported external python files

```python
def model(dbt, spark_session):
    dbt.config(
        materialized="incremental",
        incremental_strategy="merge",
        unique_key="num",
    )
    sc = spark_session.sparkContext
    sc.addPyFile("s3://athena-dbt/test/file1.py")
    sc.addPyFile("s3://athena-dbt/test/file2.py")

    def func(iterator):
        from file2 import transform

        return [transform(i) for i in iterator]

    from pyspark.sql.functions import udf
    from pyspark.sql.functions import col

    udf_with_import = udf(func)

    data = [(1, "a"), (2, "b"), (3, "c")]
    cols = ["num", "alpha"]
    df = spark_session.createDataFrame(data, cols)

    return df.withColumn("udf_test_col", udf_with_import(col("alpha")))
```

#### Known issues in python models

- Incremental models do not fully utilize spark capabilities. They depend partially on existing sql based logic which
 runs on trino.
- Snapshots materializations are not supported.
- Spark can only reference tables within the same catalog.

### Working example

seed file - employent_indicators_november_2022_csv_tables.csv

```csv
Series_reference,Period,Data_value,Suppressed
MEIM.S1WA,1999.04,80267,
MEIM.S1WA,1999.05,70803,
MEIM.S1WA,1999.06,65792,
MEIM.S1WA,1999.07,66194,
MEIM.S1WA,1999.08,67259,
MEIM.S1WA,1999.09,69691,
MEIM.S1WA,1999.1,72475,
MEIM.S1WA,1999.11,79263,
MEIM.S1WA,1999.12,86540,
MEIM.S1WA,2000.01,82552,
MEIM.S1WA,2000.02,81709,
MEIM.S1WA,2000.03,84126,
MEIM.S1WA,2000.04,77089,
MEIM.S1WA,2000.05,73811,
MEIM.S1WA,2000.06,70070,
MEIM.S1WA,2000.07,69873,
MEIM.S1WA,2000.08,71468,
MEIM.S1WA,2000.09,72462,
MEIM.S1WA,2000.1,74897,
```

model.sql

```sql
{{ config(
    materialized='table'
) }}

select row_number() over() as id
       , *
       , cast(from_unixtime(to_unixtime(now())) as timestamp(6)) as refresh_timestamp
from {{ ref('employment_indicators_november_2022_csv_tables') }}
```

timestamp strategy - model_snapshot_1

```sql
{% snapshot model_snapshot_1 %}

{{
    config(
      strategy='timestamp',
      updated_at='refresh_timestamp',
      unique_key='id'
    )
}}

select *
from {{ ref('model') }} {% endsnapshot %}
```

invalidate hard deletes - model_snapshot_2

```sql
{% snapshot model_snapshot_2 %}

{{
    config
    (
        unique_key='id',
        strategy='timestamp',
        updated_at='refresh_timestamp',
        invalidate_hard_deletes=True,
    )
}}
select *
from {{ ref('model') }} {% endsnapshot %}
```

check strategy - model_snapshot_3

```sql
{% snapshot model_snapshot_3 %}

{{
    config
    (
        unique_key='id',
        strategy='check',
        check_cols=['series_reference','data_value']
    )
}}
select *
from {{ ref('model') }} {% endsnapshot %}
```

### Snapshots Known issues

- Incremental Iceberg models - Sync all columns on schema change can't remove columns used as partitioning.
  The only way, from a dbt perspective, is to do a full-refresh of the incremental model.

- Tables, schemas and database should only be lowercase

- In order to avoid potential conflicts, make sure [`dbt-athena-adapter`](https://github.com/Tomme/dbt-athena) is not
  installed in the target environment.
  See <https://github.com/dbt-athena/dbt-athena/issues/103> for more details.

- Snapshot does not support dropping columns from the source table. If you drop a column make sure to drop the column
  from the snapshot as well. Another workaround is to NULL the column in the snapshot definition to preserve history

## Contracts

The adapter partly supports contract definition.

- Concerning the `data_type`, it is supported but needs to be adjusted for complex types. They must be specified
  entirely (for instance `array<int>`) even though they won't be checked. Indeed, as dbt recommends, we only compare
  the broader type (array, map, int, varchar). The complete definition is used in order to check that the data types
  defined in athena are ok (pre-flight check).
- the adapter does not support the constraints since no constraints don't exist in Athena.

## Contributing

See [CONTRIBUTING](CONTRIBUTING.md) for more information on how to contribute to this project.

## Contributors ✨

Thanks goes to these wonderful people ([emoji key](https://allcontributors.org/docs/en/emoji-key)):

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tbody>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/nicor88"><img src="https://avatars.githubusercontent.com/u/6278547?v=4?s=100" width="100px;" alt="nicor88"/><br /><sub><b>nicor88</b></sub></a><br /><a href="https://github.com/dbt-athena/dbt-athena/commits?author=nicor88" title="Code">💻</a> <a href="#maintenance-nicor88" title="Maintenance">🚧</a> <a href="https://github.com/dbt-athena/dbt-athena/issues?q=author%3Anicor88" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://jessedobbelae.re"><img src="https://avatars.githubusercontent.com/u/1352979?v=4?s=100" width="100px;" alt="Jesse Dobbelaere"/><br /><sub><b>Jesse Dobbelaere</b></sub></a><br /><a href="https://github.com/dbt-athena/dbt-athena/issues?q=author%3Ajessedobbelaere" title="Bug reports">🐛</a> <a href="#maintenance-jessedobbelaere" title="Maintenance">🚧</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/lemiffe"><img src="https://avatars.githubusercontent.com/u/7487772?v=4?s=100" width="100px;" alt="Lemiffe"/><br /><sub><b>Lemiffe</b></sub></a><br /><a href="#design-lemiffe" title="Design">🎨</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Jrmyy"><img src="https://avatars.githubusercontent.com/u/9251353?v=4?s=100" width="100px;" alt="Jérémy Guiselin"/><br /><sub><b>Jérémy Guiselin</b></sub></a><br /><a href="#maintenance-Jrmyy" title="Maintenance">🚧</a> <a href="https://github.com/dbt-athena/dbt-athena/commits?author=Jrmyy" title="Code">💻</a> <a href="https://github.com/dbt-athena/dbt-athena/issues?q=author%3AJrmyy" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Tomme"><img src="https://avatars.githubusercontent.com/u/932895?v=4?s=100" width="100px;" alt="Tom"/><br /><sub><b>Tom</b></sub></a><br /><a href="#maintenance-Tomme" title="Maintenance">🚧</a> <a href="https://github.com/dbt-athena/dbt-athena/commits?author=Tomme" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/mattiamatrix"><img src="https://avatars.githubusercontent.com/u/5013654?v=4?s=100" width="100px;" alt="Mattia"/><br /><sub><b>Mattia</b></sub></a><br /><a href="#maintenance-mattiamatrix" title="Maintenance">🚧</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Gatsby-Lee"><img src="https://avatars.githubusercontent.com/u/22950880?v=4?s=100" width="100px;" alt="Gatsby Lee"/><br /><sub><b>Gatsby Lee</b></sub></a><br /><a href="https://github.com/dbt-athena/dbt-athena/issues?q=author%3AGatsby-Lee" title="Bug reports">🐛</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/BrechtDeVlieger"><img src="https://avatars.githubusercontent.com/u/12074972?v=4?s=100" width="100px;" alt="BrechtDeVlieger"/><br /><sub><b>BrechtDeVlieger</b></sub></a><br /><a href="https://github.com/dbt-athena/dbt-athena/issues?q=author%3ABrechtDeVlieger" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/aartaria"><img src="https://avatars.githubusercontent.com/u/10273710?v=4?s=100" width="100px;" alt="Andrea Artaria"/><br /><sub><b>Andrea Artaria</b></sub></a><br /><a href="https://github.com/dbt-athena/dbt-athena/issues?q=author%3Aaartaria" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/maiarareinaldo"><img src="https://avatars.githubusercontent.com/u/72740386?v=4?s=100" width="100px;" alt="Maiara Reinaldo"/><br /><sub><b>Maiara Reinaldo</b></sub></a><br /><a href="https://github.com/dbt-athena/dbt-athena/issues?q=author%3Amaiarareinaldo" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/henriblancke"><img src="https://avatars.githubusercontent.com/u/1708162?v=4?s=100" width="100px;" alt="Henri Blancke"/><br /><sub><b>Henri Blancke</b></sub></a><br /><a href="https://github.com/dbt-athena/dbt-athena/commits?author=henriblancke" title="Code">💻</a> <a href="https://github.com/dbt-athena/dbt-athena/issues?q=author%3Ahenriblancke" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/svdimchenko"><img src="https://avatars.githubusercontent.com/u/39801237?v=4?s=100" width="100px;" alt="Serhii Dimchenko"/><br /><sub><b>Serhii Dimchenko</b></sub></a><br /><a href="https://github.com/dbt-athena/dbt-athena/commits?author=svdimchenko" title="Code">💻</a> <a href="https://github.com/dbt-athena/dbt-athena/issues?q=author%3Asvdimchenko" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/chrischin478"><img src="https://avatars.githubusercontent.com/u/47199426?v=4?s=100" width="100px;" alt="chrischin478"/><br /><sub><b>chrischin478</b></sub></a><br /><a href="https://github.com/dbt-athena/dbt-athena/commits?author=chrischin478" title="Code">💻</a> <a href="https://github.com/dbt-athena/dbt-athena/issues?q=author%3Achrischin478" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/sanromeo"><img src="https://avatars.githubusercontent.com/u/44975602?v=4?s=100" width="100px;" alt="Roman Korsun"/><br /><sub><b>Roman Korsun</b></sub></a><br /><a href="https://github.com/dbt-athena/dbt-athena/commits?author=sanromeo" title="Code">💻</a> <a href="https://github.com/dbt-athena/dbt-athena/issues?q=author%3Asanromeo" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Danya-Fpnk"><img src="https://avatars.githubusercontent.com/u/122433975?v=4?s=100" width="100px;" alt="DanyaF"/><br /><sub><b>DanyaF</b></sub></a><br /><a href="https://github.com/dbt-athena/dbt-athena/commits?author=Danya-Fpnk" title="Code">💻</a> <a href="https://github.com/dbt-athena/dbt-athena/issues?q=author%3ADanya-Fpnk" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/octiva"><img src="https://avatars.githubusercontent.com/u/53303191?v=4?s=100" width="100px;" alt="Spencer"/><br /><sub><b>Spencer</b></sub></a><br /><a href="https://github.com/dbt-athena/dbt-athena/commits?author=octiva" title="Code">💻</a></td>
    </tr>
  </tbody>
</table>

<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://github.com/all-contributors/all-contributors) specification.
Contributions of any kind welcome!
