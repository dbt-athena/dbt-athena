[![pypi](https://badge.fury.io/py/dbt-athena-community.svg)](https://pypi.org/project/dbt-athena-community/)
[![Imports: isort](https://img.shields.io/badge/%20imports-isort-%231674b1?style=flat&labelColor=ef8336)](https://pycqa.github.io/isort/)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
[![Stats: pepy](https://pepy.tech/badge/dbt-athena-community/month)](https://pepy.tech/project/dbt-athena-community)

# dbt-athena

* Supports dbt version `1.3.*`
* Supports [Seeds][seeds]
* Correctly detects views and their columns
* Support [incremental models][incremental]
  * On iceberg tables :
    * Support the use of `unique_key` only with query engine `v3` using the `merge` strategy
    * Support the `append` strategy
  * On Hive tables :
    * Support two incremental update strategies: `insert_overwrite` and `append`
    * Does **not** support the use of `unique_key`
* Does not support [Python models][python-models]

[seeds]: https://docs.getdbt.com/docs/building-a-dbt-project/seeds
[incremental]: https://docs.getdbt.com/docs/building-a-dbt-project/building-models/configuring-incremental-models
[python-models]: https://docs.getdbt.com/docs/build/python-models#configuring-python-models

### Installation

* `pip install dbt-athena-community`
* Or `pip install git+https://github.com/dbt-athena/dbt-athena.git`

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
- You can also use [AWS Glue](https://docs.aws.amazon.com/athena/latest/ug/glue-athena.html) to create and manage Athena databases.

### Credentials

This plugin does not accept any credentials directly. Instead, [credentials are determined automatically](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html) based on `aws cli`/`boto3` conventions and
stored login info. You can configure the AWS profile name to use via `aws_profile_name`. Checkout DBT profile configuration below for details.

### Configuring your profile

A dbt profile can be configured to run against AWS Athena using the following configuration:

| Option           | Description                                                                    | Required?   | Example               |
|------------------|--------------------------------------------------------------------------------|-------------|-----------------------|
| s3_staging_dir   | S3 location to store Athena query results and metadata                         | Required    | `s3://bucket/dbt/`    |
| s3_data_dir      | Prefix for storing tables, if different from the connection's `s3_staging_dir` | Optional    | `s3://bucket2/dbt/`   |
| s3_data_naming   | How to generate table paths in `s3_data_dir`                                   | Optional    | `schema_table_unique` |
| region_name      | AWS region of your Athena instance                                             | Required    | `eu-west-1`           |
| schema           | Specify the schema (Athena database) to build models into (lowercase **only**) | Required    | `dbt`                 |
| database         | Specify the database (Data catalog) to build models into (lowercase **only**)  | Required    | `awsdatacatalog`      |
| poll_interval    | Interval in seconds to use for polling the status of query results in Athena   | Optional    | `5`                   |
| aws_profile_name | Profile to use from your AWS shared credentials file.                          | Optional    | `my-profile`          |
| work_group       | Identifier of Athena workgroup                                                 | Optional    | `my-custom-workgroup` |
| num_retries      | Number of times to retry a failing query                                       | Optional    | `3`                   |


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
      region_name: eu-west-1
      schema: dbt
      database: awsdatacatalog
      aws_profile_name: my-profile
      work_group: my-workgroup
```

_Additional information_
* `threads` is supported
* `database` and `catalog` can be used interchangeably

### Usage notes

### Models

#### Table Configuration

* `external_location` (`default=none`)
  * If set, the full S3 path in which the table will be saved.
* `partitioned_by` (`default=none`)
  * An array list of columns by which the table will be partitioned
  * Limited to creation of 100 partitions (_currently_)
* `bucketed_by` (`default=none`)
  * An array list of columns to bucket data, ignored if using Iceberg
* `bucket_count` (`default=none`)
  * The number of buckets for bucketing your data, ignored if using Iceberg
* `format` (`default='parquet'`)
  * The data format for the table
  * Supports `ORC`, `PARQUET`, `AVRO`, `JSON`, `TEXTFILE` or `iceberg`
* `write_compression` (`default=none`)
  * The compression type to use for any storage format that allows compression to be specified. To see which options are available, check out [CREATE TABLE AS][create-table-as]
* `field_delimiter` (`default=none`)
  * Custom field delimiter, for when format is set to `TEXTFILE`
* `table_properties`: table properties to add to the table, valid for Iceberg only

#### Table location
The location in which a table is saved is determined by:

1. If `external_location` is defined, that value is used.
2. If `s3_data_dir` is defined, the path is determined by that and `s3_data_naming`
3. If `s3_data_dir` is not defined data is stored under `s3_staging_dir/tables/`

Here all the options available for `s3_data_naming`:
* `uuid`: `{s3_data_dir}/{uuid4()}/`
* `table_table`: `{s3_data_dir}/{table}/`
* `table_unique`: `{s3_data_dir}/{table}/{uuid4()}/`
* `schema_table`: `{s3_data_dir}/{schema}/{table}/`
* `s3_data_naming=schema_table_unique`: `{s3_data_dir}/{schema}/{table}/{uuid4()}/`

It's possible to set the `s3_data_naming` globally in the target profile, or overwrite the value in the table config,
or setting up the value for groups of model in dbt_project.yml


#### Incremental models

Support for [incremental models](https://docs.getdbt.com/docs/build/incremental-models).

These strategies are supported:

* `insert_overwrite` (default): The insert overwrite strategy deletes the overlapping partitions from the destination
table, and then inserts the new records from the source. This strategy depends on the `partitioned_by` keyword! If no
partitions are defined, dbt will fall back to the `append` strategy.
* `append`: Insert new records without updating, deleting or overwriting any existing data. There might be duplicate
data (e.g. great for log or historical data).
* `merge`: Conditionally updates, deletes, or inserts rows into an Iceberg table. Used in combination with `unique_key`.
Only available when using Iceberg and Athena engine version 3.

#### On schema change

`on_schema_change` is an option to reflect changes of schema in incremental models.
The following options are supported:
* `ignore` (default)
* `fail`
* `append_new_columns`
* `sync_all_columns`

In detail, please refer to [dbt docs](https://docs.getdbt.com/docs/build/incremental-models#what-if-the-columns-of-my-incremental-model-change).


#### Iceberg
The adapter supports table materialization for Iceberg.

To get started just add this as your model:
```
{{ config(
    materialized='table',
    format='iceberg',
    partitioned_by=['bucket(5, user_id)'],
    table_properties={
    	'optimize_rewrite_delete_file_threshold': '2'
    	}
) }}

SELECT
	'A' AS user_id,
	'pi' AS name,
	'active' AS status,
	17.89 AS cost,
	1 AS quantity,
	100000000 AS quantity_big,
	current_date AS my_date
```

Iceberg supports bucketing as hidden partitions, therefore use the `partitioned_by` config to add specific bucketing conditions.

It is possible to use iceberg in an incremental fashion, specifically 2 strategies are supported:
* `append`: new records are appended to the table, this can lead to duplicates
* `merge`: must be used in combination with `unique_key` and it's only available with Engine version 3.
   It performs an upsert, new record are added, and record already existing are updated


#### Unsupported functionality

Due to the nature of AWS Athena, not all core dbt functionality is supported.
The following features of dbt are not implemented on Athena:
* Snapshots

### Contributing

This connector works with Python from 3.7 to 3.10.

#### Getting started
In order to start developing on this adapter clone the repo and run this make command (see [Makefile](Makefile)) :

```bash
make setup
```

It will :
1. Install all dependencies.
2. Install pre-commit hooks.
3. Generate your `.env` file

Next, adjust `.env` file by configuring the environment variables to match your Athena development environment.

#### Running tests
You must have an AWS account with Athena setup in order to launch the tests. You can run the tests using `make`:

```bash
make test
```

### Helpful Resources

* [Athena CREATE TABLE AS](https://docs.aws.amazon.com/athena/latest/ug/create-table-as.html)
* [dbt-labs/dbt-core](https://github.com/dbt-labs/dbt-core)
* [laughingman7743/PyAthena](https://github.com/laughingman7743/PyAthena)
