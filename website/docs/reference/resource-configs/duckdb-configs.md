---
title: "DuckDB configurations"
id: "duckdb-configs"
---

## Using MotherDuck


As of `dbt-duckdb 1.5.2`, you can connect to a DuckDB instance running on [MotherDuck](https://motherduck.com) by setting your path to use an `md:` connection string, just as you would with the DuckDB CLI or the Python API.
MotherDuck databases generally work the same way as local DuckDB databases from the perspective of dbt, but there are a few differences to be aware of:
1. Currently, MotherDuck requires a specific version of DuckDB, often the latest, as specified in MotherDuck's documentation.
1. MotherDuck preloads a set of the most common DuckDB extensions for you, but does not support loading custom extensions or user-defined functions.
1. A small subset of advanced SQL features are currently unsupported; the only impact of this on the dbt adapter is that the dbt.listagg macro and foreign-key constraints will work against a local DuckDB database, but will not work against a MotherDuck database.

## Python Support

dbt added support for [Python models](https://docs.getdbt.com/docs/build/python-models) in version `1.3.0`. For most data platforms, dbt will package up the Python code defined in a `.py` file and ship it off to be executed in whatever Python environment that data platform supports (for example, Snowpark for Snowflake or Dataproc for BigQuery). 

In `dbt-duckdb`, Python models are executed in the same process that owns the connection to the DuckDB database, which by default, is the Python process that is created when you run dbt. To execute the Python model, the `.py` file that your model is defined in is teated as a Python module and loaded into the running process using [`importlib`](https://docs.python.org/3/library/importlib.html). Then construct the arguments to the model function that you defined (a dbt object that contains the names of any ref and source information your model needs and a `DuckDBPyConnection` object for you to interact with the underlying DuckDB database), call the model function, and then materialize the returned object as a table in DuckDB.

The value of the `dbt.ref` and `dbt.source` functions inside of a Python model will be a [DuckDB Relation](https://duckdb.org/docs/api/python/reference/) object that can be easily converted into a Pandas/Polars DataFrame or an Arrow table. The return value of the model function can be any Python object that DuckDB knows how to turn into a table, including a Pandas/Polars DataFrame, a DuckDB Relation, or an Arrow Table, Dataset, RecordBatchReader, or Scanner.

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

In `dbt-duckdb 1.6.0`, the profile setting `module_paths` was added which allows users to specify a list of paths on the file system that contain additional Python modules that should be added to the Python processes' `sys.path` property. This allows users to include additional helper Python modules in their dbt projects that can be accessed by running the dbt process and used to define custom dbt-duckdb plugins or library code that is helpful for creating dbt Python models.
