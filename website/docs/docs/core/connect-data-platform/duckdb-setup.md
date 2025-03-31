---
title: "DuckDB setup"
description: "Read this guide to learn about the DuckDB warehouse setup in dbt."
meta:
  maintained_by: Community
  authors: 'Josh Wills (https://github.com/jwills)'
  github_repo: 'duckdb/dbt-duckdb'
  pypi_package: 'dbt-duckdb'
  min_core_version: 'v1.0.1'
  cloud_support: Not Supported
  min_supported_version: 'DuckDB 0.3.2'
  slack_channel_name: '#db-duckdb'
  slack_channel_link: 'https://getdbt.slack.com/archives/C039D1J1LA2'
  platform_name: 'Duck DB'
  config_page: '/reference/resource-configs/no-configs'
---

:::info Community plugin

Some core functionality may be limited. If you're interested in contributing, check out the source code for each repository listed below.

:::

import SetUpPages from '/snippets/_setup-pages-intro.md';

<SetUpPages meta={frontMatter.meta} />


## Connecting to DuckDB with dbt-duckdb

[DuckDB](http://duckdb.org) is an embedded database, similar to SQLite, but designed for OLAP-style analytics instead of OLTP. The only configuration parameter that is required in your profile (in addition to `type: duckdb`) is the `path` field, which should refer to a path on your local filesystem where you would like the DuckDB database file (and it's associated write-ahead log) to be written. You can also specify the `schema` parameter if you would like to use a schema besides the default (which is called `main`).

There is also a `database` field defined in the `DuckDBCredentials` class for consistency with the parent `Credentials` class, but it defaults to `main` and setting it to be something else will likely cause strange things to happen that cannot be fully predicted, so please avoid changing it.

As of version 1.2.3, you can load any supported [DuckDB extensions](https://duckdb.org/docs/extensions/overview) by listing them in the `extensions` field in your profile. You can also set any additional [DuckDB configuration options](https://duckdb.org/docs/sql/configuration) via the `settings` field, including options that are supported in any loaded extensions. 

For example, to be able to connect to `s3` and read/write `parquet` files using an AWS access key and secret, your profile would look something like this:

<File name='profiles.yml'>

```yaml
your_profile_name:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: 'file_path/database_name.duckdb'
      extensions:
        - httpfs
        - parquet
      settings:
        s3_region: my-aws-region
        s3_access_key_id: "{{ env_var('S3_ACCESS_KEY_ID') }}"
        s3_secret_access_key: "{{ env_var('S3_SECRET_ACCESS_KEY') }}"
```

</File>

## Extensions
You can load any supported [DuckDB extensions](https://duckdb.org/docs/extensions/overview) by listing them in the `extensions` field in your profile. You can also set any additional [DuckDB configuration options](https://duckdb.org/docs/sql/configuration) via the `settings` field, including options that are supported in any loaded extensions.

As of `dbt-duckdb 1.4.1`, (experimental) support was added for DuckDB's filesystems implemented via `fsspec`. The fsspec library provides support for reading and writing files from a variety of cloud data storage systems including S3, GCS, and Azure Blob Storage. You can configure a list of fsspec-compatible implementations for use with your `dbt-duckdb` project by installing the relevant Python modules and configuring your profile like so:

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

Here, the filesystems property takes a list of configurations, where each entry must have a property named `fs` that indicates which fsspec protocol to load (for example, s3, gcs, abfs) and then an arbitrary set of other key-value pairs that are used to configure the fsspec implementation. Refer to [this example project](https://dagster.io/blog/duckdb-data-lake) that illustrates the usage of this feature to connect to a localstack instance running S3 from dbt-duckdb.

## Secret Manager

To use the [DuckDB Secrets Manager](https://duckdb.org/docs/configuration/secrets_manager.html), you can use the secrets field. For example, to connect to S3 and read/write Parquet files using an AWS access key and secret, your profile should look something like this:

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

Instead of specifying the credentials through the settings block, you can also use the `credential_chain` secret provider. This means that you can use any supported mechanism from AWS to obtain credentials (for example, web identity tokens). To use the `credential_chain` provider and automatically fetch credentials from AWS, specify the provider in the secrets key:

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

DuckDB version `0.7.0` added support for [attaching additional databases](https://duckdb.org/docs/sql/statements/attach.html) to your `dbt-duckdb` run so that you can read and write from multiple databases. Additional databases may be configured using [dbt run hooks](https://docs.getdbt.com/docs/build/hooks-operations) or via the attach argument in your profile that was added in `dbt-duckdb 1.4.0`:

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

The attached databases may be referred to in your dbt sources and models by either:

-  The basename of the database file, minus its suffix (for example `/tmp/other.duckdb` is the other database and `s3://yep/even/this/works.duckdb` is the works database). 

or 

- By an alias you specify (the `./yet/another.duckdb` database in the above configuration is referred to as `yet_another` instead of another). 

Note, these additional databases do not necessarily have to be DuckDB files. DuckDB's storage and catalog engines are pluggable. Additionally, DuckDB 0.7.0 ships with support for reading and writing from attached SQLite databases. You can indicate the type of the database you are connecting to via the type argument, which currently supports duckdb and sqlite.

## Plugins

`dbt-duckdb` has its own [plugin](https://github.com/duckdb/dbt-duckdb/blob/master/dbt/adapters/duckdb/plugins/__init__.py) system to enable advanced users to extend dbt-duckdb with additional functionality, including:
- Defining custom Python UDFs on the DuckDB database connection so that they can be used in your SQL models.
- Loading source data from Excel, Google Sheets, or SQLAlchemy tables.

You can find more details on [Writing Your Own Plugins](https://github.com/duckdb/dbt-duckdb#writing-your-own-plugins). 

To configure a plugin for use in your dbt project, use the `plugins` property on the profile:

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

Every plugin must have a module property that indicates where the plugin class to load is defined. There are a [set of built-in plugins](https://github.com/duckdb/dbt-duckdb/blob/master/dbt/adapters/duckdb/plugins) you can define, that may be referenced by their base filename (`excel` or `gsheet`), while user-defined plugins (which are described later in this document) should be referred to by their full module path name (such as, a `lib.my.custom` module that defines a class named plugin.)

Each plugin instance has a name for logging and reference purposes that defaults to the name of the module but that may be overridden by the user by setting the alias property in the configuration. Finally, modules may be initialized using an arbitrary set of key-value pairs that are defined in the config dictionary. In this example, the gsheet plugin is initialized with the setting method: oauth and the `sqlalchemy` plugin (aliased as "sql") is initialized with a `connection_url` that is set as an environment variable.

Note, using plugins may require you to add additional dependencies to the Python environment that your dbt-duckdb pipeline runs in:

- `excel` depends on `pandas`, and `openpyxl` or `xlsxwriter` to perform writes
- `gsheet` depends on `gspread` and `pandas`
- `iceberg` depends on `pyiceberg` and `Python >= 3.8`
- `sqlalchemy` depends on `pandas`, `sqlalchemy`, and the driver(s) you need

## External Files

One of DuckDB's most powerful features is its ability to read and write CSV, JSON, and Parquet files directly, without needing to import/export them from the database first.

### Reading from external files

You may reference external files in your dbt models either directly or as dbt sources by configuring the `external_location` in either the meta or the config option on the source definition. The difference is that the settings under the meta option will be propagated to the documentation for the source generated in dbt docs generate, but the settings under the config option will not be. Any source settings that should be excluded from the docs should be specified by the config, while any options that you would like to be included in the generated documentation should live under meta.

```yml
sources:
  - name: external_source
    meta:
      external_location: "s3://my-bucket/my-sources/{name}.parquet"
    tables:
      - name: source1
      - name: source2
```

Here, the meta options on external_source defines `external_location` as an f-string that allows us to express a pattern that indicates the location of any of the tables defined for that source. 

So a dbt model like:

```sql
SELECT *
FROM {{ source('external_source', 'source1') }}
```

will be compiled as:

```sql
SELECT *
FROM 's3://my-bucket/my-sources/source1.parquet'
```

If one of the source tables deviates from the pattern or needs some other special handling, then the `external_location` can also be set on the meta options for the table itself, for example:

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

In this situation, the `external_location` setting on the source2 table will take precedence. 

A dbt model like:

```sql
SELECT *
FROM {{ source('external_source', 'source2') }}
```

will be compiled to the SQL query:

```sql
SELECT *
FROM read_parquet(['s3://my-bucket/my-sources/source2a.parquet', 's3://my-bucket/my-sources/source2b.parquet'])
```

Note that the value of the `external_location` property does not need to be a path-like string; it can also be a function call, which is helpful in the case that you have an external source that is a CSV file which requires special handling for DuckDB to load it correctly:

```yml
sources:
  - name: flights_source
    tables:
      - name: flights
        config:
          external_location: "read_csv('flights.csv', types={'FlightDate': 'DATE'}, names=['FlightDate', 'UniqueCarrier'])"
          formatter: oldstyle
```

Note, you will need to override the default `str.format` string formatting strategy for this example because the `types={'FlightDate': 'DATE'}` argument to the `read_csv` function will be interpreted by `str.format` as a template to be matched on. This will cause a `KeyError: "'FlightDate'"` when there's an attempt to parse the source in a dbt model. 

The formatter configuration option for the source indicates whether you should use newstyle string formatting (the default), oldstyle string formatting, or template string formatting. 

You can read up on the strategies for different string formatting techniques and find examples, in this [discussion of dbt-duckdb's integration test](https://stackoverflow.com/questions/78821149/formatting-strings-in-dask-duckdb).

You can also create dbt models that are backed by external files via the external materialization strategy:

```sql 
{{
  config(materialized='external', location='local/directory/file.parquet') 
}}

SELECT m.*, s.id IS NOT NULL as has_source_id
FROM {{ ref('upstream_model') }} m
LEFT JOIN {{ source('upstream', 'source') }} s USING (id)
```


| Option | Default | Description |
| --- | --- | --- |
| location | `external_location` macro | The path to write the external materialization to. See below for more details. |
| format | parquet |The format of the external file |(parquet, csv, or json).
| delimiter | , | For CSV files, the delimiter to use for fields. |
| options | None | Any other options to pass to DuckDB's COPY operation (for example partition_by, codec, etc). |
| `glue_register` | false | If true, try to register the file created by this model with the AWS Glue Catalog. |
| `glue_database` | default | The name of the AWS Glue database to register the model with. |


If the location argument is specified, it must be a filename (or S3 bucket/path), and `dbt-duckdb` will attempt to infer the format argument from the file extension of the location if the format argument is unspecified (this functionality was added in version 1.4.1.)

If the location argument is not specified, then the external file will be named after the `model.sql` (or `model.py`) file that defined it with an extension that matches the format argument (parquet, csv, or json). By default, the external files are created relative to the current working directory, but you can change the default directory (or S3 bucket/prefix) by specifying the `external_root` setting in your DuckDB profile.

`dbt-duckdb` supports the `delete+insert` and `append` [strategies](/docs/build/incremental-strategy#built-in-strategies) for incremental table models, but there's no support for incremental materialization strategies for external models.

### Registering External Models

When using `:memory:` as the DuckDB database, subsequent dbt runs can fail when selecting a subset of models that depend on external tables. This is because external files are only registered as DuckDB views when they are created, not when they are referenced. To overcome this issue, use the `register_upstream_external_models` macro that can be triggered at the beginning of a run. To enable this automatic registration, place the following in your `dbt_project.yml` file:


```yml
on-run-start:
  - "{{ register_upstream_external_models() }}"
```
