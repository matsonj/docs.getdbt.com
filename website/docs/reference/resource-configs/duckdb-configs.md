---
title: "DuckDB configurations"
id: "duckdb-configs"
---


## Profile

A super-minimal dbt-duckdb profile only needs one setting:

```yml
default:
  outputs:
    dev:
      type: duckdb
  target: dev
```

This will run your dbt-duckdb pipeline against an in-memory DuckDB database that will not be persisted after your run completes.

To have your dbt pipeline persist relations in a DuckDB file, set the path field in your profile to the path of the DuckDB file that you would like to read and write on your local filesystem. (If the path is not specified, the path is automatically set to the special value `:memory:` and the database will run in-memory, without persistence).


`dbt-duckdb adds` the `database` property: its value is automatically set to the basename of the file in the path argument with the suffix removed. For example, if the path is `/tmp/a/dbfile.duckdb`, the `database` field will be set to `dbfile`.


## Using MotherDuck


As of `dbt-duckdb 1.5.2`, you can connect to a DuckDB instance running on MotherDuck by setting your path to use a md: connection string, just as you would with the DuckDB CLI or the Python API.
MotherDuck databases generally work the same way as local DuckDB databases from the perspective of dbt, but there are a few differences to be aware of:
1. Currently, MotherDuck requires a specific version of DuckDB, often the latest, as specified in MotherDuck's documentation
2. MotherDuck databases do not support transactions, so there is a new `disable_transactions` profile option that will be automatically enabled if you are connecting to a MotherDuck database in your path.
3. MotherDuck preloads a set of the most common DuckDB extensions for you, but does not support loading custom extensions or user-defined functions.
4. A small subset of advanced SQL features are currently unsupported; the only impact of this on the dbt adapter is that the dbt.listagg macro and foreign-key constraints will work against a local DuckDB database, but will not work against a MotherDuck database.

## Extensions
You can load any supported DuckDB extensions by listing them in the `extensions` field in your profile. You can also set any additional DuckDB configuration options via the `settings` field, including options that are supported in any loaded extensions.

As of `dbt-duckdb 1.4.1`, we have added (experimental!) support for DuckDB's (experimental!) support for filesystems implemented via `fsspec`. The fsspec library provides support for reading and writing files from a variety of cloud data storage systems including S3, GCS, and Azure Blob Storage. You can configure a list of fsspec-compatible implementations for use with your `dbt-duckdb` project by installing the relevant Python modules and configuring your profile like so:

```yml
default:
  outputs:
    dev:
      type: duckdb
      path: /tmp/dbt.duckdb
      filesystems:
        - fs: s3
          anon: false
          key: "{{ env_var('S3_ACCESS_KEY_ID') }}"
          secret: "{{ env_var('S3_SECRET_ACCESS_KEY') }}"
          client_kwargs:
            endpoint_url: "http://localhost:4566"
  target: dev
```

Here, the filesystems property takes a list of configurations, where each entry must have a property named fs that indicates which fsspec protocol to load (so s3, gcs, abfs, etc.) and then an arbitrary set of other key-value pairs that are used to configure the fsspec implementation. You can see a simple example project that illustrates the usage of this feature to connect to a Localstack instance running S3 from dbt-duckdb here.

## Secret Manager

To use the DuckDB Secrets Manager, you can use the secrets field. For example, to be able to connect to S3 and read/write Parquet files using an AWS access key and secret, your profile would look something like this:

```yml
default:
  outputs:
    dev:
      type: duckdb
      path: /tmp/dbt.duckdb
      extensions:
        - httpfs
        - parquet
      secrets:
        - type: s3
          region: my-aws-region
          key_id: "{{ env_var('S3_ACCESS_KEY_ID') }}"
          secret: "{{ env_var('S3_SECRET_ACCESS_KEY') }}"
  target: dev
```

### Fetching credentials from context

Instead of specifying the credentials through the settings block, you can also use the CREDENTIAL_CHAIN secret provider. This means that you can use any supported mechanism from AWS to obtain credentials (e.g., web identity tokens). You can read more about the secret providers here. To use the CREDENTIAL_CHAIN provider and automatically fetch credentials from AWS, specify the provider in the secrets key:

```yml
default:
  outputs:
    dev:
      type: duckdb
      path: /tmp/dbt.duckdb
      extensions:
        - httpfs
        - parquet
      secrets:
        - type: s3
          provider: credential_chain
  target: dev
```

## Attaching Additional Databases

DuckDB version `0.7.0` added support for attaching additional databases to your `dbt-duckdb` run so that you can read and write from multiple databases. Additional databases may be configured using dbt run hooks or via the attach argument in your profile that was added in `dbt-duckdb 1.4.0`:

```yml
default:
  outputs:
    dev:
      type: duckdb
      path: /tmp/dbt.duckdb
      attach:
        - path: /tmp/other.duckdb
        - path: ./yet/another.duckdb
          alias: yet_another
        - path: s3://yep/even/this/works.duckdb
          read_only: true
        - path: sqlite.db
          type: sqlite
```

The attached databases may be referred to in your dbt sources and models by either the basename of the database file minus its suffix (e.g., /tmp/other.duckdb is the other database and s3://yep/even/this/works.duckdb is the works database) or by an alias that you specify (so the ./yet/another.duckdb database in the above configuration is referred to as yet_another instead of another.) Note that these additional databases do not necessarily have to be DuckDB files: DuckDB's storage and catalog engines are pluggable, and DuckDB 0.7.0 ships with support for reading and writing from attached SQLite databases. You can indicate the type of the database you are connecting to via the type argument, which currently supports duckdb and sqlite.

## Plugins

`dbt-duckdb` has its own plugin system to enable advanced users to extend dbt-duckdb with additional functionality, including:
- Defining custom Python UDFs on the DuckDB database connection so that they can be used in your SQL models
- Loading source data from Excel, Google Sheets, or SQLAlchemy tables

You can find more details on how to write your own plugins here. To configure a plugin for use in your dbt project, use the `plugins` property on the profile:

```yml
default:
  outputs:
    dev:
      type: duckdb
      path: /tmp/dbt.duckdb
      plugins:
        - module: gsheet
          config:
            method: oauth
        - module: sqlalchemy
          alias: sql
          config:
            connection_url: "{{ env_var('DBT_ENV_SECRET_SQLALCHEMY_URI') }}"
        - module: path.to.custom_udf_module
```

Every plugin must have a module property that indicates where the Plugin class to load is defined. There is a set of built-in plugins that are defined in dbt.adapters.duckdb.plugins that may be referenced by their base filename (e.g., excel or gsheet), while user-defined plugins (which are described later in this document) should be referred to via their full module path name (e.g. a lib.my.custom module that defines a class named Plugin.)

Each plugin instance has a name for logging and reference purposes that defaults to the name of the module but that may be overridden by the user by setting the alias property in the configuration. Finally, modules may be initialized using an arbitrary set of key-value pairs that are defined in the config dictionary. In this example, we initialize the gsheet plugin with the setting method: oauth and we initialize the sqlalchemy plugin (aliased as "sql") with a connection_url that is set via an environment variable.
Please remember that using plugins may require you to add additional dependencies to the Python environment that your dbt-duckdb pipeline runs in:
- excel depends on pandas, and openpyxl or xlsxwriter to perform writes
- gsheet depends on gspread and pandas
- iceberg depends on pyiceberg and Python >= 3.8
- sqlalchemy depends on pandas, sqlalchemy, and the driver(s) you need

## Python Support
dbt added support for Python models in version `1.3.0`. For most data platforms, dbt will package up the Python code defined in a .py file and ship it off to be executed in whatever Python environment that data platform supports (e.g., Snowpark for Snowflake or Dataproc for BigQuery.) In dbt-duckdb, we execute Python models in the same process that owns the connection to the DuckDB database, which by default, is the Python process that is created when you run dbt. To execute the Python model, we treat the .py file that your model is defined in as a Python module and load it into the running process using importlib. We then construct the arguments to the model function that you defined (a dbt object that contains the names of any ref and source information your model needs and a DuckDBPyConnection object for you to interact with the underlying DuckDB database), call the model function, and then materialize the returned object as a table in DuckDB.

The value of the dbt.ref and dbt.source functions inside of a Python model will be a DuckDB Relation object that can be easily converted into a Pandas/Polars DataFrame or an Arrow table. The return value of the model function can be any Python object that DuckDB knows how to turn into a table, including a Pandas/Polars DataFrame, a DuckDB Relation, or an Arrow Table, Dataset, RecordBatchReader, or Scanner.

### Batch Processing

As of version 1.6.1, it is possible to both read and write data in chunks, which allows for larger-than-memory datasets to be manipulated in Python models. Here is a basic example:

```py
import pyarrow as pa

def batcher(batch_reader: pa.RecordBatchReader):
    for batch in batch_reader:
        df = batch.to_pandas()
        # Do some operations on the DF...
        # ...then yield back a new batch
        yield pa.RecordBatch.from_pandas(df)

def model(dbt, session):
    big_model = dbt.ref("big_model")
    batch_reader = big_model.record_batch(100_000)
    batch_iter = batcher(batch_reader)
    return pa.RecordBatchReader.from_batches(batch_reader.schema, batch_iter)
```

### Using Local Python Modules

In dbt-duckdb 1.6.0, we added a new profile setting named module_paths that allows users to specify a list of paths on the filesystem that contain additional Python modules that should be added to the Python processes' sys.path property. This allows users to include additional helper Python modules in their dbt projects that can be accessed by the running dbt process and used to define custom dbt-duckdb Plugins or library code that is helpful for creating dbt Python models.

## External Files

One of DuckDB's most powerful features is its ability to read and write CSV, JSON, and Parquet files directly, without needing to import/export them from the database first.

### Reading from external files

You may reference external files in your dbt models either directly or as dbt sources by configuring the external_location in either the meta or the config option on the source definition. The difference is that settings under the meta option will be propagated to the documentation for the source generated via dbt docs generate, but the settings under the config option will not be. Any source settings that should be excluded from the docs should be specified via config, while any options that you would like to be included in the generated documentation should live under meta.

```yml
sources:
  - name: external_source
    meta:
      external_location: "s3://my-bucket/my-sources/{name}.parquet"
    tables:
      - name: source1
      - name: source2
```

Here, the meta options on external_source defines external_location as an f-string that allows us to express a pattern that indicates the location of any of the tables defined for that source. So a dbt model like:

```sql
SELECT *
FROM {{ source('external_source', 'source1') }}
```

will be compiled as:

```sql
SELECT *
FROM 's3://my-bucket/my-sources/source1.parquet'
```

If one of the source tables deviates from the pattern or needs some other special handling, then the external_location can also be set on the meta options for the table itself, for example:

```yml
sources:
  - name: external_source
    meta:
      external_location: "s3://my-bucket/my-sources/{name}.parquet"
    tables:
      - name: source1
      - name: source2
        config:
          external_location: "read_parquet(['s3://my-bucket/my-sources/source2a.parquet', 's3://my-bucket/my-sources/source2b.parquet'])"
```

In this situation, the external_location setting on the source2 table will take precedence, so a dbt model like:

```sql
SELECT *
FROM {{ source('external_source', 'source2') }}
```

will be compiled to the SQL query:

```sql
SELECT *
FROM read_parquet(['s3://my-bucket/my-sources/source2a.parquet', 's3://my-bucket/my-sources/source2b.parquet'])
```

Note that the value of the external_location property does not need to be a path-like string; it can also be a function call, which is helpful in the case that you have an external source that is a CSV file which requires special handling for DuckDB to load it correctly:

```yml
sources:
  - name: flights_source
    tables:
      - name: flights
        config:
          external_location: "read_csv('flights.csv', types={'FlightDate': 'DATE'}, names=['FlightDate', 'UniqueCarrier'])"
          formatter: oldstyle
```

Note that we need to override the default str.format string formatting strategy for this example because the `types={'FlightDate': 'DATE'}` argument to the read_csv function will be interpreted by str.format as a template to be matched on, which will cause a `KeyError: "'FlightDate'"` when we attempt to parse the source in a dbt model. The formatter configuration option for the source indicates whether we should use newstyle string formatting (the default), oldstyle string formatting, or template string formatting. You can read up on the strategies the various string formatting techniques use at this Stack Overflow answer and see examples of their use in this dbt-duckdb integration test.
Writing to external files
We support creating dbt models that are backed by external files via the external materialization strategy:

```sql 
{{ config(materialized='external', location='local/directory/file.parquet') }}

SELECT m.*, s.id IS NOT NULL as has_source_id
FROM {{ ref('upstream_model') }} m
LEFT JOIN {{ source('upstream', 'source') }} s USING (id)
```


Option
Default
Description
location
external_location macro
The path to write the external materialization to. See below for more details.
format
parquet
The format of the external file (parquet, csv, or json)
delimiter
,
For CSV files, the delimiter to use for fields.
options
None
Any other options to pass to DuckDB's COPY operation (e.g., partition_by, codec, etc.)
glue_register
false
If true, try to register the file created by this model with the AWS Glue Catalog.
glue_database
default
The name of the AWS Glue database to register the model with.

If the location argument is specified, it must be a filename (or S3 bucket/path), and dbt-duckdb will attempt to infer the format argument from the file extension of the location if the format argument is unspecified (this functionality was added in version 1.4.1.)

If the location argument is not specified, then the external file will be named after the model.sql (or model.py) file that defined it with an extension that matches the format argument (parquet, csv, or json). By default, the external files are created relative to the current working directory, but you can change the default directory (or S3 bucket/prefix) by specifying the external_root setting in your DuckDB profile.

dbt-duckdb supports the delete+insert and append strategies for incremental table models, but unfortunately it does not yet support incremental materialization strategies for external models.

### Registering External Models

When using :memory: as the DuckDB database, subsequent dbt runs can fail when selecting a subset of models that depend on external tables. This is because external files are only registered as DuckDB views when they are created, not when they are referenced. To overcome this issue we have provided the register_upstream_external_models macro that can be triggered at the beginning of a run. To enable this automatic registration, place the following in your dbt_project.yml file:


```yml
on-run-start:
  - "{{ register_upstream_external_models() }}"
```


