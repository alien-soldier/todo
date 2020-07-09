#Apache Arrow
[Source](https://arrow.apache.org/docs/python/)
>Apache Arrow is a development platform for in-memory analytics. It contains a set of technologies that enable big data systems to process and move data fast.   
The project is developing a multi-language collection of libraries for solving systems problems related to in-memory analytical data processing. This includes such topics as:  
>*   Reading and writing file formats (like CSV, Apache ORC, and Apache Parquet)
>*   In-memory analytics and query processing

###Python bindings: Python API of Apache Arrow, PyArrow
>The Arrow Python bindings (also named **“PyArrow”**) have first-class integration with NumPy, pandas, and built-in Python objects. 
They are based on the C++ implementation of Arrow.

**NumPy Integration**
>PyArrow allows converting back and forth from **NumPy arrays** to **Arrow Arrays**.    
Methods: `pyarrow.array()` and `to_numpy()`
```python
import numpy as np
import pyarrow as pa

# NumPy array to Arrow 
data = np.arange(10, dtype='int16')
arr = pa.array(data) 

# Arrow to NumPy array 
arr = pa.array([4, 5, 6], type=pa.int32())
view = arr.to_numpy()
```

**Pandas Integration**
>To interface with pandas, PyArrow provides various conversion routines to consume pandas structures and convert back to them.  
**DataFrames**     
The equivalent to a pandas DataFrame in Arrow is a Table. Both consist of a set of named columns of equal length. 
While pandas only supports flat columns, the Table also provides nested columns, 
thus it can represent more data than a DataFrame, so a full conversion is not always possible.     

>Conversion from a Table to a DataFrame is done by calling pyarrow.Table.to_pandas(). 
The inverse is then achieved by using pyarrow.Table.from_pandas().     
Methods: `pyarrow.Table.to_pandas()` and `pyarrow.Table.from_pandas()` 
```python
import pyarrow as pa
import pandas as pd

df = pd.DataFrame({"a": [1, 2, 3]})
# Convert from pandas to Arrow
table = pa.Table.from_pandas(df)
# Convert back to pandas
df_new = table.to_pandas()

# Infer Arrow schema from pandas
schema = pa.Schema.from_pandas(df)
```
*Handling pandas Indexes*
>Methods like pyarrow.Table.from_pandas() have a preserve_index option which defines how to preserve (store) or not to preserve (to not store)  
The default of preserve_index is None, which behaves as follows:    
>*   RangeIndex is stored as metadata-only, not requiring any extra storage.    
>*   Other index types are stored as one or more physical data columns in the resulting Table.
>
>**To not store the index at all pass preserve_index=False**    
>**To force all index data to be serialized in the resulting table, pass preserve_index=True.**

**Memory Usage and Zero Copy**
>Reducing Memory Use in Table.to_pandas:    
>`self_destruct=True`, this destroys the internal Arrow memory buffers in each column Table object as they are converted to the pandas-compatible representation,
```python
df = table.to_pandas(self_destruct=True)
del table  # not necessary, but a good practice
```
##Apache Parquet
###Reading and Writing the Apache Parquet Format
>The Apache Parquet project provides a standardized open-source columnar storage format for use in data analysis systems.    
[Source](https://parquet.apache.org/documentation/latest/)   
Apache Arrow is an ideal in-memory transport layer for data that is being read or written with Parquet files        
PyArrow includes Python bindings to parquet also, which thus enables reading and writing Parquet files with pandas as well.     

*   Writing: **[Pandas] -> [Arrow Table] -> [Parquet]**    
*   Reading: **[Parquet] -> [Arrow Table] -> [Pandas]**    

**Obtaining pyarrow with Parquet Support**  
>If you installed pyarrow with pip or conda, it should be built with Parquet support bundled:
`import pyarrow.parquet as pq`

###Reading and Writing Single Files
>The functions `read_table()` and `write_table()` read and write the pyarrow.Table objects, respectively.   
```python
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq

df = pd.DataFrame(...)

table = pa.Table.from_pandas(df)

pq.write_table(table, 'example.parquet')
```
This creates a single Parquet file. In practice, a Parquet dataset may consist of many files in many directories. 
We can read a single file back with `read_table`:
```python
table = pq.read_table('example.parquet')

table.to_pandas()
```
You can pass a subset of columns to read, which can be much faster than reading the whole file (due to the columnar layout):    
`pq.read_table('example.parquet', columns=['one', 'three'])`

When reading a subset of columns from a file that is going to be a dataFrame, we can use read_pandas to maintain any additional index column data:
```python
pq.read_pandas('example.parquet', columns=['two']).to_pandas()
```
>**read_pandas** docstring says:    
Read a Table from Parquet format, also reading DataFrame index values if known in the file metadata.

In above function source We need not use a string to specify the origin of the file. It can be any of:
*   A file path as a string
*   A NativeFile from PyArrow like [BufferReader, BufferOutputStream, CompressedInputStream , ...](https://arrow.apache.org/docs/python/memory.html#io-native-file)
*   A Python file object (like open() handler)

**Parquet file writing options**
>write_table() has a number of options to control various settings when writing a Parquet file.     
>`data_page_size`: to control the approximate size of encoded data pages within a column chunk. This currently defaults to 1MB

**Finer-grained Reading and Writing**
>read_table uses the **ParquetFile** class, which has other features:
```python
parquet_file = pq.ParquetFile('example.parquet')
parquet_file.metadata

    num_columns: 4
    num_rows: 3
    num_row_groups: 1
    serialized_size: 2637

parquet_file.schema

    required group field_id=0 schema {
          optional int32 field_id=1 A;
          optional int32 field_id=2 B;
          optional int64 field_id=3 __index_level_0__;
    }
```
>A Parquet file consists of multiple row groups.    
`read_table` will read all of the row groups and concatenate them into a single table. 
You can read individual row groups with read_row_group:
```python
parquet_file.num_row_groups
2
table = parquet_file.read_row_group(0)
```
We can similarly write a Parquet file with multiple row groups by using ParquetWriter:
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

Other points:
*   The read_dictionary option in read_table and ParquetDataset will cause columns to be read as DictionaryArray, 
which will become pandas.Categorical when converted to pandas.
*   `coerce_timestamps`   option to allow you to select the desired resolution for timestamp.
*   Parquet implementations use dictionary encoding when writing files. (like RDBMS, Repeated values map to keys).   
if the dictionaries grow too large, then they became slow. Whether dictionary encoding is used can be toggled using the use_dictionary option:
 `pq.write_table(table, where, use_dictionary=False)`
*   The data pages within a column in a row group can be compressed after the encoding passes (dictionary, RLE encoding). 
In PyArrow we use Snappy compression by default, but `Brotli`, `Gzip`, and `uncompressed` are also supported:   
```python
pq.write_table(table, where, compression='snappy')
pq.write_table(table, where, compression='gzip')
pq.write_table(table, where, compression='brotli')
pq.write_table(table, where, compression='none')
```
###`Snappy` generally results in better performance, while `Gzip` may yield smaller files.
>These settings can also be set on a per-column basis:
```python
pq.write_table(table, where, compression={'foo': 'snappy', 'bar': 'gzip'}, use_dictionary=['foo', 'bar'])
```
####Partitioned Datasets (Multiple Files)
A dataset partitioned by year and month may look like on disk:
```dataset_name/
  year=2007/
    month=01/
       0.parq
       1.parq
       ...
    month=02/
       0.parq
       1.parq
       ...
  year=2008/
    month=01/
    ...
  ...
```
**Writing to Partitioned Datasets**     
Multiple Parquet files constitute a Parquet dataset. These may present in a number of ways: 
*   A **list** of Parquet absolute file paths
*   A **directory name** containing nested directories defining a partitioned dataset
```python
pq.write_to_dataset(table, root_path='dataset_name', partition_cols=['one', 'two'])
```
Columns are partitioned in the order they are given.    
The partition splits are determined by the unique values in the partition columns.

**Reading from Partitioned Datasets**   
The ParquetDataset class accepts either a **directory name** or a **list or file paths**, and can discover and infer some common partition structures
```python
dataset = pq.ParquetDataset('dataset_name/')
table = dataset.read()
```

You can also use the convenience function read_table exposed by **pyarrow.parquet** that avoids the need for an 
additional Dataset object creation step.    
```table = pq.read_table('dataset_name')```

>####Note
>**The ParquetDataset is being reimplemented based on the new generic Dataset API "Tabular Datasets"**
This is not yet the default, but can already be enabled by passing the `use_legacy_dataset=False` keyword to `ParquetDataset` or `read_table()`
>```python
>dataset = pq.ParquetDataset('dataset_name/', use_legacy_dataset=False)
>table = dataset.read()
># or
>table = pq.read_table('dataset_name/', use_legacy_dataset=False)
>```
>Enabling this gives the following new features:
>*   Filtering on all columns (using row group statistics) instead of only on the partition keys.
>In the future, this will be turned on by default.


##Filtering
>Parquet supports **predicate pushdown ** on RowGroup and Page level. While page-level filtering is not yet available 
>and will be a bit more complex to implement, you can already do basic RowGroup filtering using pyarrow.    
>With `pyarrow.ParquetFile(…).metadata.row_group(…).column(…).{min,max}` you get the minimum and maximum of a RowGroup for a given column.  
>When you compare your filters against these statistics, you should be able to only read a subset of the file using `read_rowgroup`.      

But this is not the useful solution.    
####What should I use?
[API Source for parquet.read_table()](https://arrow.apache.org/docs/python/generated/pyarrow.parquet.read_table.html#pyarrow.parquet.read_table)    
[API Source for parquet.parquet.ParquetDataset()](https://arrow.apache.org/docs/python/generated/pyarrow.parquet.ParquetDataset.html#pyarrow.parquet.ParquetDataset)

Both of "read single parquet file" and "read dataset" has a `filters` parameter.

`pyarrow.parquet.read_table()` and `pyarrow.parquet.ParquetDataset()`  
>**filters** (List[Tuple] or List[List[Tuple]] or None (default)):  
>Rows which do not match the filter predicate will be removed from scanned data. 
>Partition keys embedded in a nested directory structure will be exploited to avoid loading files at all if they contain no matching rows.      
>If `use_legacy_dataset` is True, filters can only reference partition keys and only a hive-style directory structure is supported.         
>When setting `use_legacy_dataset` to False, also within-file level filtering and different partitioning schemes are supported.     
>Predicates are expressed in disjunctive normal form (DNF)   
>`[ [('x', '=', 0), ...], ...]`    
>The innermost tuples each describe a single column predicate. The list of inner predicates is interpreted as a conjunction (AND)   
>The most outer list combines these filters as a disjunction (OR).  
>`[ [('col1', '=', 0), ('col2', '>' 10)], [('col3' > 20)] ]`  
>Its pandas meaning is:     
>`Mask = ((df['col1'] == 0) & (df['col2] > 10)) | (df['col3'] > 20)`

[Parquet file Api summary](https://arrow.apache.org/docs/python/api/formats.html)
-   `ParquetDataset(path_or_paths[, filesystem, …])`               
-   `ParquetFile(source[, metadata, …])`      
-   `ParquetWriter(where, schema[, filesystem, …])`       
-   `read_table(source[, columns, use_threads, …])`       
-   `read_metadata(where[, memory_map])`
-   `read_pandas(source[, columns, use_threads, …])`
-   `read_schema(where[, memory_map])`
-   `write_metadata(schema, where[, version, …])`
-   `write_table(table, where[, row_group_size, …])`
-   `write_to_dataset(table, root_path[, …])`


##What should I do?
>1.  write you parquet file in logical part (row group)
>2.  use Filters for fetching the part you want

#Writing
####How to write parquet in logical part?
*  **Using `parquet.writer`**    
[Sourcce](http://peter-hoffmann.com/2020/understand-predicate-pushdown-on-rowgroup-level-in-parquet-with-pyarrow-and-python.html)
split your pandas on favorite part   
dfs = [df1, df2, ...]
```python
table = pa.Table.from_pandas(dfs[0], preserve_index=False)
writer = pq.ParquetWriter('file.parquet', table.schema)

for df in dfs:
    table = pa.Table.from_pandas(df, preserve_index=False)
    writer.write_table(table)
writer.close()
```
*  **Using Pandas method**  
Pandas implements parquet api using pyarrow or fastparquet engine internally     

>`df.to_parquet` takes **kwargs and pass them to parquet writer api  
for example in `df.to_parquet("filename.parquet", row_group_size=500, engine="pyarrow")`, 
**`row_group_size`** parameter chunks dataFrame to specified chucked row group and pass them on iterator to write which is not useful
[source](Control row groups with pandas.DataFrame.to_parquet)

>But **fastparquet** engine takes a `row_group_offsets` parameter which can take your favorite indexes as a list       
[Api source](https://fastparquet.readthedocs.io/en/latest/api.html?highlight=row_group#fastparquet.ParquetFile.iter_row_groups)
```python
df.to_parquet("df2.parquet", row_group_offsets=10000000, engine="fastparquet")
df.to_parquet("df2.parquet", row_group_offsets=[0, 500], engine="fastparquet")
```
>`row_group_offsets`: If int, row-groups will be ***approximately*** this many rows, rounded down to make row groups about the same size; 
>if a list, the explicit index values to start new row groups.

If you want to preserve the index pass `preserve_index=True` or `index=True`    
Remember Since an entire row group might need to be read, your logically splitting is so important. 
 
Before paying more attention to fastparquet let see filtering on reading
 
#Reading (with Filtering) 
[](https://github.com/pandas-dev/pandas/issues/26551)

[Pyarrow all parquet api](https://arrow.apache.org/docs/python/api/formats.html)    
[Pandas read parquet](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.read_parquet.html)
```python
# Method 1, Using parquet API
import pyarrow.parquet as pq
# Read a Table from Parquet format 
table = pq.read_table('df.parquet', columns=[], use_legacy_dataset=False, filters=[])
df = table.to_pandas()

# Method 2, Using parquet API
# Read a Table from Parquet format, also reading DataFrame index values if known in the file metadata 
# important: For this method parquet should have index 
df = pq.read_pandas('df.parquet', columns=[], use_legacy_dataset=False, filters=[]).to_pandas()


# Method 3, Using Pandas API
# pandas.read_parquet(path, engine: str = 'auto', columns=None, **kwargs)
import pandas as pd
df = pd.read_parquet('df.parquet', engine='pyarrow', use_legacy_dataset=False, filters=[])
```
In Aabove pandas method, we see **kwargs,   
**Kwargs ars Any additional kwargs are passed to the engine.
[Important Question](https://github.com/pandas-dev/pandas/issues/26551)
Someone answers above question: "pyarrow would need to expose this; pandas already passes thru kwargs"
But it has some problem with pandas 1.0.4 and you should use `pandas 1.0.3` and `pyarrow '0.17.1'`

---
##fastparquet
A Python interface to the Parquet file format.  
read and write Parquet files, in single- or multiple-file format. The latter is commonly found in hive/Spark usage. 
ability to choose row divisions and partitioning on write.

**Reading**     
>To open and read the contents of a Parquet file:
```python
from fastparquet import ParquetFile
pf = ParquetFile('myfile.parq')
df = pf.to_pandas()
```
The Pandas data-frame, df will contain all columns in the target file, and all row-groups concatenated together.    
If the data is a multi-file collection `“_metadata”` file has meta-data associated with the data.   
The properties columns, count, dtypes and statistics are available in`“_metadata”`.

You may specify which columns to load, which of those to keep as categoricals (if the data uses dictionary encoding), 
and which column to use as the pandas index. By selecting columns, we only access parts of the file, and efficiently skip 
columns that are not of interest.   
```python
df2 = pf.to_pandas(['col1', 'col2'], categories=['col1'])
# or
df2 = pf.to_pandas(['col1', 'col2'], categories={'col1': 12})
```

***row-groups can be skipped by providing a list of filters.    
Note that only row-groups that have no data at all meeting the specified requirements will be skipped.
This means that all rows of a row group which meets the condition fetch from file as result.    
This is in contrast with **pyarrow.parquet** filters which returns just the interested rows.***
```python
from fastparquet import ParquetFile
pf = ParquetFile('outfile')
# returns entire rows of each row group which has one A == 99  
df = pf.to_pandas(filters=[("A", "==", 99)])

# returns just the rows with A == 99 
import pyarrow.parquet as pq
df = pq.read_pandas('outfile', use_legacy_dataset=False, filters=[("A", ">", 98)]).to_pandas()
print(df)
```

**Writing**     
>To create a single Parquet file from a dataframe:
```python
from fastparquet import write
write('outfile.parq', df)
```
The function write provides a number of options.

*   the compression algorithms (typically “snappy”, for fast, but not too space-efficient), which can vary by column
*   the row-group splits to apply, which may lead to efficiencies on loading, if some row-groups can be skipped. 
Statistics (min/max) are calculated for each column in each row-group on the fly.
*   multi-file saving can be enabled with the file_scheme keyword: hive-style output is a directory with a single 
metadata file and several data-files. ***(This is partitioning)****
```python
write('outfile2.parq', df, row_group_offsets=[0, 10000, 20000], compression='GZIP', file_scheme='hive')
```

**Partitions and row-groups**
The Parquet format allows for partitioning the data by the values of some (low-cardinality) columns and by row sequence number.
>Splitting on both row-groups and partitions can potentially result in many data-files and large metadata. 
>It should be used sparingly, when partial selecting of the data is anticipated.

**Use both partitioning and row_group splitting just when partial selecting is a routine**
**Row groups**
>The keyword parameter row_group_offsets allows control of the row sequence-wise splits in the data. For example, 
>with the default value, each row group will contain 50 million rows. 
>The exact index of the start of each row-group can also be specified, 
>which may be appropriate in the presence of a monotonic index: such as a time index might lead to the desire to have all the row-group boundaries coincide with year boundaries in the data.

**Partitions**
>In the presence of some low-cardinality columns, it may be advantageous to split data data on the values of those columns. 
>This is done by writing a directory structure with key=value names. Multiple partition columns can be chosen, 
>leading to a multi-level directory tree.

Consider the following directory tree from this Spark example:
```
table/
    gender=male/
        country=US/
            data.parquet
        country=CN/
            data.parquet

    gender=female/
        country=US/
            data.parquet
        country=CN/
            data.parquet
```

>Here the two partitioned fields are gender and country, each of which have two possible values, resulting in four datafiles. 
>The corresponding columns are not stored in the data-files, but inferred on load, so space is saved, and if selecting 
>based on these values, potentially some of the data need not be loaded at all.     
If there were two row groups and the same partitions as above, each leaf directory would contain (up to) two files, 
>for a total of eight. If a row-group happens to contain no data for one of the field value combinations, that data file is omitted. 


**Iteration**
>For data-sets too big to fit conveniently into memory, it is possible to iterate through the row-groups in a similar way to reading by chunks from CSV with pandas.
```python
pf = ParquetFile('myfile.parq')
for df in pf.iter_row_groups():
    print(df.shape)
    # process sub-data-frame df
```
>Thus only one row-group is in memory at a time. The same set of options are available as in to_pandas allowing, 
>for instance, reading only specific columns, loading to categoricals or to ignore some row-groups using filtering.     
To get the first row-group only, one would go:      
`first = next(iter(pf.iter_row_groups()))`

Good Doc: [fastparquet Api](https://fastparquet.readthedocs.io/en/latest/api.html)

>**`fastparquet.write()`**  
>**file_scheme**: **‘simple’|’hive’**   
If simple: all goes in a single file    
If hive: **each row group is in a separate file**, and a separate file (called “_metadata”) contains the metadata.  
>
>***This is for both partitioning and row group?***  
>In other word you can have partition_ed data but in one file (simple)
and Not partion_ed file with multiple row_group which Composed of multiple files. (hive)

>**partition_on**: list of column names     
Passed to groupby in order to split data within each row-group, producing a structured directory tree.   
>Note: as with pandas, null values will be dropped. Ignored if file_scheme is simple.     

>**append**: bool (False)   
If False, construct data-set from scratch; if True, add new row-group(s) to existing data-set.      
In the latter case, the data-set must exist, and the schema must match the input data.










```python
import pandas as pd
import pyarrow.parquet as pq

pq_file = pq.ParquetFile('file.parquet')
print(pq_file.metadata.num_columns)
print(pq_file.metadata.num_rows)
print(pq_file.metadata.num_row_groups)
        
schema = pq_file.metadata.schema
data = [[schema.column(i).name, schema.column(i).physical_type, schema.column(i).logical_type] for i in range(len(schema))]

schema_df = pd.DataFrame(data, columns=['column', 'physical', 'logical'])
print(schema_df)


data = []
for row_g in range(pq_file.metadata.num_row_groups):
    row_g_meta = pq_file.metadata.row_group(row_g)
    data.append([row_g, row_g_meta.num_rows, row_g_meta.total_byte_size])

row_g_size_stats = pd.DataFrame(data, columns=['row_group', 'rows', 'size'])
print(row_g_size_stats)


# For watching stats of a row_group 
rg_meta = pq_file.metadata.row_group(0) 
print(rg_meta.column(0))  # pass column index from schema_df, starting from Zero, shows info like:
# path_in_schema: column_name
# is_stats_set: 
# has_min_max: 
# min: 
# max: 
# null_count: 
# distinct_count: 
# num_values: 

# If you logically split row group based on col_1, you can see min, max each row group
# Looking at the min and max statistics of the col_1:
column = 1 # index of column
data = []
for row_g in range(pq_file.metadata.num_row_groups):
    row_g_meta = pq_file.metadata.row_group(row_g)
    data.append([row_g, str(row_g_meta.column(column).statistics.min), str(row_g_meta.column(column).statistics.max)])

row_g_boundary_stats = pd.DataFrame(data, columns=["row_group", "min", "max"])
print(row_g_boundary_stats)
```
[Source](http://peter-hoffmann.com/2020/understand-predicate-pushdown-on-rowgroup-level-in-parquet-with-pyarrow-and-python.html)




----
#### Some docs about iterating on row_group and append mode of fastparquet remains to explain. 

---
#### Below lines was some old stuff which needs to review for deleting or merging with above docs. 


#*pyarrow*

##*Dumping*

###*Directly From Pandas To Parquet*
>`df.to_parquet("df.parquet", row_group_size=10000, engine="pyarrow", index=True)`  
>row_group_size splits dataFrame to row_group. It does not accept offset. Then rational splitting on same size can't solve problem.  




##*Fetching*








*   NumbaDeprecationWarning: The 'numba.jitclass' decorator has moved to 'numba.experimental.jitclass' to better reflect the experimental nature of the functionality. Please update your imports to accommodate this change and see http://numba.pydata.org/numba-doc/latest/reference/deprecation.html#change-of-jitclass-location for the time frame. Numpy 8 = numba.jitclass(spec 8)(Numpy)
    >Answer:
    The version on conda-forge includes this fix
    `conda install -c conda-forge fastparquet`  
    [Answer Source](https://github.com/dask/fastparquet/issues/495)

*   RemoveError: 'setuptools' is a dependency of conda and cannot be removed from conda's operating environment.                                                                                                                                                                                                                                                                                                                                                                                                      
During `conda install -c conda-forge fastparquet`
    >Answer:    
    `conda update --force conda`    
    [Answer Source](https://stackoverflow.com/questions/57549872/removeerror-setuptools-is-a-dependency-of-conda-and-cannot-be-removed-from-co)     
    
