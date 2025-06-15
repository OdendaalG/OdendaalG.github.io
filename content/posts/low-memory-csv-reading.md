+++
date = '2025-03-03T08:06:59Z'
draft = true
title = 'Low Memory CSV Reading'
tags = ['python', 'pandas', 'memory', 'csv']
+++

# Introduction
At a client where we were tasked with building a data manipulation tool based off of JSON configs , we encountered an issue
with memory usage and pandas. We use pandas in this case to read all the CSV files into dataframes and export them to
python dictionaries.

Additionally, when the python memory allocation usage tests indicate something like `2.8GB` of memory used, the virtual machine in Azure's Container Instance Docker indicates `6GB` of memory used.

# Background

# Methodology and Design
In this test today we will be attempting four different methods alongside the baseline that is currently implemented. As is
engineering standard practice, we aim to isolate as much as possible and alter as little as possible from run to run. Each
run will also consist of three runs, averaging the results to get a more accurate measurement.

## Tools to be used
As we will be measuring memory usage in a python application, we will use the package `tracemalloc` to measure how much
memory the entire application uses, what packages and files are using the most, and finally we will use a recursive function
to measure the actual memory usage of the output.

[`tracemalloc`](https://docs.python.org/3/library/tracemalloc.html) will be used in the following manner:
```python
import tracemalloc
tracemalloc.start()
tracemalloc.get_traced_memory() # first measurement
# function called here
tracemalloc.get_traced_memory() # final measurement

snapshot = tracemalloc.take_snapshot().statistics('traceback') # used to get the top 5 largest memory objects
```

`get_traced_memory` will give us an indication of the entire application's memory usage and the snapshot will tell us which
packages and files are using the most memory.

An additional function that is being used to measure the ultimate `output` size is as
follows (thanklessly stolen from an LLM) (see appendix).

Methods to try:
1. Standard pandas read_csv
2. `records` output instead of `dict`
3. Garbage collection
4. Chunking and garbage collection
5. Raw read and dictionary creation
6. `records` and garbage collection
7. `records` and garbage collection with chunking

Measurements to be taken:
- Total memory usage before and after
- Top 5 largest memory objects from tracemalloc's snapshots

We will be reading in eight CSV files, each about 14Mb of data.

Finally, code snippets will show core code, not try-catch and error handling, nor will it show any references to additional
functions created.

## Standard pandas read_csv
Straight pandas read_csv is used for this step:
```python
# Snippet from the read_csv module of the orchestration code
from pandas import read_csv
# ...

output = []
for file in files:
    df = read_csv(file)
    data = df.to_dict(orient='dict')
    output.append(data)

return output
```
This approach simply uses the 'read_csv' function from pandas and will be the baseline.

## `records` output instead of `dict`
When we convert the dataframe to a dictionary to pass onto the next step, we can use the `records` output instead of `dict`:
```python
# Snippet from the read_csv module of the orchestration code
from pandas import read_csv
# ...

output = []
for file in files:
    df = read_csv(file)
    data = df.to_dict(orient='records')
    output.append(data)

return output
```
This approach uses the `records` output instead of `dict` to pass onto the next step.

## Garbage Collection
```python
# Snippet from the read_csv module of the orchestration code
from pandas import read_csv
import gc
# ...

output = []
for file in files:
    df = read_csv(file)
    data = df.to_dict(orient='dict')
    del df
    gc.collect()
    output.append(data)

return output
```
In addition to using the straight `read_csv` function from pandas, we're making sure to delete the dataframe reference
and perform garbage collection.

## Chunking and Garbage Collection
```python
# Snippet from the read_csv module of the orchestration code
from pandas import read_csv
import gc
# ...

output = []
for file in files:
    chunksize = 10000
    for chunk in pd.read_csv(file, chunksize=chunksize):
        data = chunk.to_dict(orient='dict')
        del chunk
        gc.collect()
        output.append(data)

return output
```

## Raw Read and Dictionary Creation
This will remove the need for pandas and garbage collection.

```python
# Snippet from the read_csv module of the orchestration code
import csv
import gc
# ...

output = []
for file in files:
    with open(file, 'r') as f:
        reader = csv.DictReader(f)
        data = list(reader)
        output.append(data)

return output
```

# Results

Using the `get_size_recursive` function, we have found that the issue might not be related to the use of python, but rather to

The size given by the recursive function is:
- dict orientation: `3198.274093 MB`
- records orientation: `2195.308373 MB`

Table 1: Comparison of Memory Usage

| Method | Memory Usage (one file, current) | Memory Usage (one file, peak) | Memory Usage (eight files, current) | Memory Usage (eight files, peak) |
| --- | --- | --- | --- | --- |
| Standard pandas read_csv | 397 MB | 440 MB | 3199 MB | 3243 MB |
| `records` output instead of `dict` | 311.09 MB | 311.09 MB | 2238.44 MB | 2238.44 MB |
| Garbage Collection | 397 MB | 440 MB | 3199 MB | 3243 MB |
| Chunking and Garbage Collection | 378 MB | 380 MB | 3084 MB | 3084 MB |
| Raw Read and Dictionary Creation | 432.74 MB | 432.77 MB | 3523.89 MB | 3523.92 MB |
| `records` and Garbage Collection | 270.29 MB | 311.09 MB | 2195.92 MB | 2238.43 MB |
| `records` and Garbage Collection with chunking | 280.83 MB | 280.86 MB | 2275.49 MB | 2275.84 MB |



# Appendices and Code
## Recursive Size Function
```python
import sys

def get_size_recursive(obj, seen=None):
    """Recursively find the total size of an object and all its contents in bytes"""
    if seen is None:
        seen = set()

    # If we've already seen the object, don't count it again
    obj_id = id(obj)
    if obj_id in seen:
        return 0

    # Mark this object as seen
    seen.add(obj_id)
    size = sys.getsizeof(obj)

    # If object is a dict, add size of keys and values
    if isinstance(obj, dict):
        size += sum(get_size_recursive(k, seen) + get_size_recursive(v, seen)
                   for k, v in obj.items())

    # If object is a list or tuple, add size of elements
    elif isinstance(obj, (list, tuple)):
        size += sum(get_size_recursive(item, seen) for item in obj)

    # If object has __dict__ attribute, add size of attributes
    elif hasattr(obj, '__dict__'):
        size += get_size_recursive(obj.__dict__, seen)

    # If object has __slots__, add size of slots
    elif hasattr(obj, '__slots__'):
        size += sum(get_size_recursive(getattr(obj, slot, None), seen)
                   for slot in obj.__slots__ if hasattr(obj, slot))

    return size
```
