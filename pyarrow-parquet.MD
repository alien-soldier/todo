
[Overview off Pyarrow parquet api](https://blog.datasyndrome.com/python-and-parquet-performance-e71da65269ce)   
[About Filtering](https://stackoverflow.com/questions/56522977/using-predicates-to-filter-rows-from-pyarrow-parquet-parquetdataset/64394428#64394428)  
[About RowGroup](http://peter-hoffmann.com/2020/understand-predicate-pushdown-on-rowgroup-level-in-parquet-with-pyarrow-and-python.html)  
[Pyarrow itself explaining](https://www.mikulskibartosz.name/how-to-write-parquet-file-in-python/)    
[Pyarrow Parquet Official API](https://arrow.apache.org/docs/python/api/formats.html#parquet-files)  

## Apache Arrow

Apache Arrow defines columnar array data structures. These data structures are exposed in Python through a series of
interrelated classes:

*   **Type Metadata**:
        Instances of pyarrow.DataType, which describe a logical array type
*   **Schemas:**
        Instances of pyarrow.Schema, which describe a named collection of types. These can be thought of as the column
        types in a table-like object.
*   **Arrays:**
        Instances of pyarrow.Array, which are atomic, contiguous columnar data structures composed from Arrow Buffer objects
*   **Record Batches**:
        Instances of pyarrow.RecordBatch, which are a collection of Array objects with a particular Schema
*   **Tables:**
        Instances of pyarrow.Table, a logical table data structure in which each column consists of one or more
        pyarrow.Array objects of the same type.

[A good doc for explaining above concepts:](https://www.mikulskibartosz.name/how-to-write-parquet-file-in-python/)  
    
    In this article, I am going to show you how to define a Parquet schema in Python, how to manually prepare a Parquet table
    and write it to a file, how to convert a Pandas data frame into a Parquet table.

    1. Defining a schema
        Column types can be automatically inferred, but for the sake of completeness, I am going to define the schema.

        import pyarrow as pa
        subscription_schema = pa.schema([
                                            ('timestamp', pa.timestamp('ms')),
                                            ('id', pa.int32()),
                                            ('email', pa.string())
                                        ])
    2. Columns and batches
        A batch is a collection of equal-length arrays. Every array contains data of a single column.
        Those columns are aggregated into a batch using the schema we have just defined.

        In my example, I will store three values in every column. Here are the values. One more time, note that I don’t
        need to specify the type explicitly.

        timestamps = pa.array([
                                datetime(2019, 9, 3, 9, 0, 0),
                                datetime(2019, 9, 3, 10, 0, 0),
                                datetime(2019, 9, 3, 11, 0, 0)
                            ], type = pa.timestamp('ms'))

        ids = pa.array([1, 2, 3], type = pa.int32())

        emails = pa.array(
                            ['first@example.com', 'second@example.com', 'third@example.com'],
                            type = pa.string()
                        )

        batch = pa.RecordBatch.from_arrays(
                                            [timestamps, ids, emails],
                                            names = subscription_schema
                                         )
    3.  Tables
        We use a Table to define a single logical dataset. It can consist of multiple batches.
        A table is a structure that can be written to a file using the write_table function.

            import pyarrow.parquet as pq
            table = pa.Table.from_batches([batch])
            pq.write_table(table, 'test/subscriptions.parquet')

        When I call the write_table function, it will write a single parquet file called subscriptions.parquet into the
        “test” directory in the current working directory.

    Writing Pandas data frames
    We can define the same data as a Pandas data frame.

    dataframe = pd.DataFrame([
                                [datetime(2019, 9, 3, 9, 0, 0), 1, 'first@example.com'],
                                [datetime(2019, 9, 3, 10, 0, 0), 1, 'second@example.com'],
                                [datetime(2019, 9, 3, 11, 0, 0), 1, 'third@example.com'],
                            ], columns = ['timestamp', 'id', 'email'])

    When the data frame is ready, we can use the from_pandas function to convert the data frame into a table.
    Such a table can be written into a file in exactly the same way as in the previous example.

    table_from_pandas = pa.Table.from_pandas(dataframe)
    pq.write_table(table_from_pandas, 'test/subscriptions_pandas.parquet')
  
**Metadata**        
>Type Metadata include:  
>*   Fixed-length primitive types: numbers, booleans, date and times, fixed size binary, decimals, and other     
            values that fit into a given number
>*   Variable-length primitive types: binary, string     
>*   Nested types: list, struct, and union       
>*   Dictionary type: An encoded categorical type (more on this later)          
>
>Each logical data type in Arrow has a corresponding factory function for creating an instance of that type object in Python:

```python
import pyarrow as pa
    t1 = pa.int32()
    print(ti)
    Out: DataType(int32)
```

>We use the name logical type because the physical storage may be the same for one or more types.
For example, int64, float64, and timestamp[ms] all occupy 64 bits per value.

These objects are metadata; they are used for describing the data in arrays, schemas, and record batches.

The 'Field' type is a type plus a name and optional user-defined metadata:
   ```python
 f0 = pa.field('int32_field', t1)
```
>
>
>
>
>
>
    
**struct**  
>A 'struct' is a collection of named fields:

```python
fields = [
                pa.field('s0', t1),
                pa.field('s1', t2),
                pa.field('s2', t4),
                pa.field('s3', t6),
              ]
```

**Schema**  
>The 'Schema' type is similar to the struct array type; it defines the column names and types in a record batch or table data structure.

**Arrays**  
>For each data type, there is an accompanying array data structure for holding memory buffers that define a single
contiguous chunk of columnar array data.

**Tables**
>The PyArrow Table type is not part of the Apache Arrow specification, but is rather a tool to help with wrangling
multiple record batches and array pieces as a single logical dataset. As a relevant example, we may receive multiple
small record batches in a socket stream, then need to concatenate them into contiguous memory for use in NumPy or pandas.

```python
batches = [batch] * 5
table = pa.Table.from_batches(batches)
```

---
### Here is starting point    
As you’ll see in the pandas section, we can convert these objects (Array and Table) to contiguous NumPy arrays for use in pandas


**NumPy Integration**   
    PyArrow allows converting back and forth from NumPy arrays to Arrow Arrays.

```python
    # NumPy to Arrow
    data = np.arange(10, dtype='int16')
    arr = pa.array(data)

    # Arrow to NumPy
    # This is limited to primitive types for which NumPy has the same physical representation as Arrow, and assuming
    # the Arrow data has no nulls.
    arr = pa.array([4, 5, 6], type=pa.int32())
    view = arr.to_numpy()
```

**Pandas Integration**  
    To interface with pandas, PyArrow provides various conversion routines to consume pandas structures and convert back to them.
    (Pandas Integration support for null values)    

**DataFrames**
>The equivalent to a pandas DataFrame in Arrow is a Table.

```python
 # Convert from pandas to Arrow
table = pa.Table.from_pandas(df)

# Convert back to pandas
df_new = table.to_pandas()

# Infer Arrow schema from pandas
schema = pa.Schema.from_pandas(df)
```

**Series**  
In Arrow, the most similar structure to a pandas Series is an Array.
You can convert a pandas Series to an Arrow Array using pyarrow.Array.from_pandas().
As Arrow Arrays are always nullable, you can supply an optional mask using the mask parameter to mark all null-entries.

By default pyarrow tries to preserve and restore the .index data as accurately as possible.

**Handling pandas Indexes:**
>Methods like pyarrow.Table.from_pandas() have a preserve_index option which defines how to preserve (store) or
not to preserve (to not store) the data in the index member of the corresponding pandas object.
This data is tracked using schema-level metadata in the internal arrow::Schema object.

>The default of preserve_index is None, which behaves as follows:   
>*   RangeIndex is stored as metadata-only, not requiring any extra storage.
>*   Other index types are stored as one or more physical data columns in the resulting Table
>
>   To not store the index at all pass preserve_index=False. Since storing a RangeIndex can cause issues in some
limited scenarios (such as storing multiple DataFrame objects in a Parquet file), to force all index data to be
serialized in the resulting table, pass preserve_index=True.


**Type differences**    
>With the current design of pandas and Arrow, it is not possible to convert all column types unmodified.
One of the main issues here is that pandas has no support for nullable columns of arbitrary type.
Also datetime64 is currently fixed to nanosecond resolution.

pandas -> Arrow Conversion and vise-versa. look the below table     
[Conversion DataTypes Table](https://arrow.apache.org/docs/python/pandas.html#type-differences)

**Zero Copy Series Conversions**    
>To try to limit the potential effects of “memory doubling” during Table.to_pandas, we provide a couple of options: 
>*  split_blocks=True
>*  self_destruct=True  
   
```python
# Used together
df = table.to_pandas(split_blocks=True, self_destruct=True)
del table  # not necessary, but a good practice     
```
>
>   above call will yield significantly lower memory usage in some scenarios. Without these options, to_pandas will always double memory.
    Note that self_destruct=True is not guaranteed to save memory.

**Arrow/Pandas Timestamps**     
>Arrow timestamps are stored as a 64-bit integer with column metadata to associate a time unit (e.g. milliseconds,
microseconds, or nanoseconds), and an optional time zone. Pandas (Timestamp) uses a 64-bit integer representing nanoseconds and an optional time zone.

---
# Parquet
The Apache Parquet project provides a standardized open-source columnar storage     
[source](https://arrow.apache.org/docs/python/api/formats.html#parquet-files)

## Writing Single File

1.  #### The function write_table()  
    `write_table(table, where[, row_group_size, …])`  ->   Write a pyarrow.Table to Parquet format.

    ```python
    import pyarrow.parquet as pq
    pq.write_table(table, 'example.parquet')
    ```

2.  #### Class ParquetWriter for logically controlling row groups (writing to multiple row groups)   
    `ParquetWriter(where, schema[, filesystem, …])`   ->  Class for incrementally building a Parquet file for Arrow tables.

    >Methods:   
    >*   __init__(where, schema[, filesystem, …])    
    >*    write_table(table[, row_group_size])       
    >*    close()

    ```python
    writer = pq.ParquetWriter('example2.parquet', table.schema)
    for i in range(3):
        writer.write_table(table)
    writer.close()
    ```

    Alternatively python with syntax can also be use:
    ```python
    with pq.ParquetWriter('example3.parquet', table.schema) as writer:
        for i in range(3):
            writer.write_table(table)
    ```

    **Partitioned Datasets (Multiple Files)**

    >Multiple Parquet files constitute a Parquet dataset. These may present in a number of ways:
    >*   A list of Parquet absolute file paths
    >*   A directory name containing nested directories defining a partitioned dataset   
    The root path in this case specifies the parent directory to which data will be saved. The partition columns are
    the column names by which to partition the dataset. Columns are partitioned in the order they are given.
    The partition splits are determined by the unique values in the partition columns.

**Writing to Partitioned Datasets**

1.  #### The function write_to_dataset()     
    `write_to_dataset(table, root_path[, …])` ->  Wrapper around parquet.write_table for writing a Table to Parquet format by partitions.

```python
pq.write_to_dataset(table, root_path='dataset_name', partition_cols=['one', 'two'], **kwargs)
```
**kwargs -> (dict,) – Additional kwargs for passing to write_table() function


## Reading Single or multiple Files

1.  #### The functions read_table()      
    `read_table(source[, columns, use_threads, …])`   ->  Read a Table from Parquet format  
    >some important params:
    >*   source :  can be a single file name or directory name.
    >*   partitioning : The partitioning scheme for a partitioned dataset. The default of “hive” assumes directory names with key=value pairs
    >*   filters : Rows which do not match the filter predicate will be removed from scanned data.
    >*   use_pandas_metadata:  If True and file has custom pandas schema metadata, ensure that index columns are also loaded.

    ***calling read_pandas() is equal to passing use_pandas_metadata=True***

     ```python
    table2 = pq.read_table('example.parquet', columns=['one', 'three'])
    ```

2.  #### The function read_pandas()
    `read_pandas(source[, columns])`  ->  Read a Table from Parquet format, also reading DataFrame index values if known in the file metadata   

    When reading a subset of columns from a file that used a Pandas dataframe as the source, we use read_pandas
    to maintain any additional index column data:

    ```python
    pq.read_pandas('example.parquet', columns=['two']).to_pandas()
    ```

**read_table()** uses the ParquetFile class, which has other features:

>   ParquetFile(source[, metadata, …])  ->  Reader interface for a single Parquet file.     
>>Methods    
>*  __init__(source[, metadata, …])
>*  iter_batches([batch_size, row_groups, …])           ->  Read streaming batches from a Parquet file
>*  read([columns, use_threads, use_pandas_metadata])   ->  Read a Table from Parquet format,
>*  read_row_group(i[, columns, use_threads, …])        ->   Read a single row group from a Parquet file.
>*  read_row_groups(row_groups[, columns, …])           ->   Read a multiple row groups from a Parquet file.
>*  scan_contents([columns, batch_size])                ->  Read contents of file for the given columns and batch size.         
>
>>Attributes
>*  metadata
>*  num_row_groups
>*  schema  ->  Return the Parquet schema, unconverted to Arrow types
>*  schema_arrow    ->  Return the inferred Arrow schema, converted from the whole Parquet file’s schema

```python
parquet_file = pq.ParquetFile('example.parquet')
parquet_file.metadata
parquet_file.schema
parquet_file.num_row_groups
parquet_file.read_row_group(0)
```

read_table will read all of the row groups and concatenate them into a single table. You can read individual

**Reading from Partitioned Datasets**

3.  #### Class ParquetDataset
    `([path_or_paths, filesystem, …])`  ->  Encapsulates details of reading a complete Parquet  
    dataset possibly consisting of multiple files and partitions in subdirectories.

    The ParquetDataset class accepts either a directory name or a list or file paths, and can discover and infer
    some common partition structures, such as those produced by Hive:
    
    ```python
    dataset = pq.ParquetDataset('dataset_name/')
    table = dataset.read()
    ```

    You can also use the convenience function read_table exposed by pyarrow.parquet that avoids the need for
    an additional Dataset object creation step.
    ```python
    table = pq.read_table('dataset_name')
    ```



**Inspecting the Parquet File Metadata:**

1. ## The read_metadata fucntion
    `read_metadata(where[, memory_map])`   ->  Read FileMetadata from footer of a single Parquet file.

    The FileMetaData of a Parquet file can be accessed through ParquetFile class as:

    ```python
    parquet_file = pq.ParquetFile('example.parquet')
    metadata = parquet_file.metadata
    ```
    
    or can also be read directly using read_metadata():
    
    ```python
    metadata = pq.read_metadata('example.parquet')
    ```

    The returned FileMetaData object allows to inspect the Parquet file metadata, such as the row groups and column chunk metadata and statistics:


## Tabular Datasets
https://arrow.apache.org/docs/python/dataset.html#dataset

>The pyarrow.dataset module provides functionality to efficiently work with tabular,and multi-file datasets.
This includes:
>*   A unified interface that supports different sources and file formats (Parquet, Feather / Arrow IPC, and CSV files)and different file systems (local, cloud).
>*   Discovery of sources (crawling directories, handle directory-based partitioned datasets, basic schema normalization, ..)
>*   Optimized reading with predicate pushdown (filtering rows), projection (selecting and deriving columns),
    and optionally parallel reading

For those familiar with the existing pyarrow.parquet.ParquetDataset for reading Parquet datasets: pyarrow.dataset’s
goal is similar but not specific to the Parquet format and not tied to Python:



