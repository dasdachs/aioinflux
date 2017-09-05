# aioinflux

Asynchronous Python client for InfluxDB. 
Built on top of [`aiohttp`](https://github.com/aio-libs/aiohttp) and 
[`asyncio`](https://docs.python.org/3/library/asyncio.html).

InfluxDB is an open-source distributed time series database.
Find more about InfluxDB at http://influxdata.com/


## Installation

Aioinflux is not yet listed on PyPI. Install directly from sources:

    # Latest stable version
    $ pip install git+https://github.com/plugaai/aioinflux
    
    # Latest development commit
    $ pip install git+https://github.com/plugaai/aioinflux@dev


### Dependencies

Aioinflux supports Python 3.6+ **ONLY**. For older Python versions please use the 
[official Python client](https://github.com/influxdata/influxdb-python) 

Third-party library dependencies are: [`aiohttp`](https://github.com/aio-libs/aiohttp) 
for all HTTP request handling and [`pandas`](https://github.com/pandas-dev/pandas) for 
`DataFrame` reading/writing support.


## Usage

### TL;DR:

This sums most of what you can do with `aioinflux`:

```python
import asyncio
from aioinflux import AsyncInfluxDBClient

point = dict(time='2009-11-10T23:00:00Z',
             measurement='cpu_load_short',
             tags={'host': 'server01',
                   'region': 'us-west'},
             fields={'value': 0.64})

client = AsyncInfluxDBClient(db='testdb')

coros = [client.create_database(db='testdb'),
         client.write(point),
         client.query('SELECT value FROM cpu_load_short')]
         
loop = asyncio.get_event_loop() 
results = loop.run_until_complete(asyncio.gather(*coros))
for result in results:
    print(result)
```

### Client modes

Despite its name, `AsyncInfluxDBClient` can also run in sync/blocking modes. 
Avialable modes are: `async` (default), `blocking` and `dataframe`.

Example using `blocking` mode:

```python
client = AsyncInfluxDBClient(db='testdb', mode='blocking')
client.ping()
client.write(point)
client.query('SELECT value FROM cpu_load_short')
```

See [Retrieving DataFrames](#retrieving-dataframes) for `dataframe` mode usage.

### Writing data

Input data can be:
1) A string properly formatted in InfluxDB's line protocol
2) A dictionary containing the following keys: `measurement`, `time`, `tags`, `fields`
3) A Pandas DataFrame with a DatetimeIndex
4) An iterable of one of the above

Input data in formats 2-4 are parsed into the 
[line protocol](https://docs.influxdata.com/influxdb/v1.3/write_protocols/line_protocol_reference/) 
before being written to InfluxDB. 
All parsing functionality is located at [`serialization.py`](aioinflux/serialization.py).
Beware that serialization is not highly optimized (PRs are welcome!) and may become a bottleneck depending 
on your application.


The `write` method returns `True` when successful and raises an `InfluxDBError` otherwise.  


#### Writing dictionary-like objects

Aioinflux accepts any dictionary-like object (mapping) as input. However, that dictionary must 
be properly formated and contain the following keys:

1) **`measurement`**: Optional. Must be a string-like object. If ommited, must be specified when 
  calling `AsyncInfluxDBClient.write` by passing a `measurement` argument.
1) **`time`**: Optional. The value can be `datetime.datetime`, date-like string 
  (e.g., `2017-01-01`, `2009-11-10T23:00:00Z`) or anything else that can be parsed by Pandas 
  [`Timestamp`](https://pandas.pydata.org/pandas-docs/stable/timeseries.html) class.
1) **`tags`**: Optional. This must contain another mapping of field names and values.
  Both tag keys and values should be strings.  
1) **`fields`**: Mandatory. This must contain another mapping of field names and values.
  Field keys should be strings. Field values can be `float`, `int`, `str`, or `bool` or any equivalent type. 

Any fields other then the above will be ignored when writing data to InfluxDB.


A typical dictionary-like point would look something like the following:

```python
{'time': '2009-11-10T23:00:00Z',
'measurement': 'cpu_load_short',
'tags': {'host': 'server01', 'region': 'us-west'},
'fields': {'value1': 0.64, 'value2': True, 'value3': 10}}
```


#### Writing DataFrames

Aioinflux also accepts Pandas dataframes as input. The only requirements for the dataframe is
that the **index must be of type `DatetimeIndex`**. Also, any column whose `dtype` is `object` will be 
converted to a string representation.

A typical DataFrame input should look something like the following:

```text
                                       LUY       BEM       AJW tag
2017-06-24 08:45:17.929097+00:00  2.545409  5.173134  5.532397   B
2017-06-24 10:15:17.929097+00:00 -0.306673 -1.132941 -2.130625   E
2017-06-24 11:45:17.929097+00:00  0.894738 -0.561979 -1.487940   B
2017-06-24 13:15:17.929097+00:00 -1.799512 -1.722805 -2.308823   D
2017-06-24 14:45:17.929097+00:00  0.390137 -0.016709 -0.667895   E
```

The measurement name must be specified with the `measurement` argument when calling `AsyncInfluxDBClient.write`.
Additional tags can also be passed using arbitrary keyword arguments.
 
**Example:**

```python
client = AsyncInfluxDBClient(db='testdb', mode='blocking')
client.write(df, measurement='prices', tag_columns=['tag'], asset_class='equities')
```

In the example above, `df` is the DataFrame we are trying to write to InfluxDB and 
`measurement` is the measurement we are writing to. 

`tag_columns` is in an optional iterable 
telling which of the dataframe columns should be parsed as tag values. If `tag_columns` is 
not explitly passed, all columns in the dataframe will be treated as InfluxDB field values.

Any other keyword arguments passed to `AsyncInfluxDBClient.write` are treated as extra tags 
which will be attached to the data being written to InfluxDB. Any string which is a valid 
[InfluxDB identifier](https://docs.influxdata.com/influxdb/v1.3/query_language/spec/#identifiers) 
and valid [Python identifier](https://docs.python.org/3/reference/lexical_analysis.html#identifiers) 
can be used as an extra tag key (with the exception of they strings `data`, `measurement` and `tag_columns`).

See `AsyncInfluxDBClient.write` docstring for details.


### Querying data

Querying data is as simple as passing an InfluxDB query string to `AsyncInfluxDBClient.write`:

```python
client.query('SELECT myfield FROM mymeasurement')
```

The result (in `blocking` and `async` modes) is a dictionary containing the raw JSON data 
returned by the InfluxDB 
[HTTP API](https://docs.influxdata.com/influxdb/v1.3/guides/querying_data/#querying-data-using-the-http-api):

```python
{'results': [{'series': [{'columns': ['time', 'Price', 'Volume'],
     'name': 'mymeasurement',
     'values': [[1491963424224703000, 5783, 100],
      [1491963424375146000, 5783, 200],
      [1491963428374895000, 5783, 100],
      [1491963429645478000, 5783, 1100],
      [1491963429655289000, 5783, 100],
      [1491963437084443000, 5783, 100],
      [1491963442274656000, 5783, 900],
      [1491963442274657000, 5782, 5500],
      [1491963442274658000, 5781, 3200],
      [1491963442314710000, 5782, 100]]}],
   'statement_id': 0}]}
```


#### Retrieving DataFrames

```text
                                  Price  Volume
2017-04-12 02:17:04.224703+00:00   5783     100
2017-04-12 02:17:04.375146+00:00   5783     200
2017-04-12 02:17:08.374895+00:00   5783     100
2017-04-12 02:17:09.645478+00:00   5783    1100
2017-04-12 02:17:09.655289+00:00   5783     100
2017-04-12 02:17:17.084443+00:00   5783     100
2017-04-12 02:17:22.274656+00:00   5783     900
2017-04-12 02:17:22.274657+00:00   5782    5500
2017-04-12 02:17:22.274658+00:00   5781    3200
2017-04-12 02:17:22.314710+00:00   5782     100
```

#### Chunked responses

TODO

#### Convenience functions

Aioinflux provides some wrappers around ``AsyncInfluxDBClient.query`` in order to provide
convenient access to commonly used query patterns. Appropriate named arguments must be 
passed (e.g.: `db`, `measurement`, etc).

Examples:

```python
client.create_database(db='foo')
client.drop_measurement(measurement='bar')
client.show_users()
```
For more complex queries, pass a raw query to ``AsyncInfluxDBClient.query``.

Please refer to the [source](aioinflux/client.py#L199) for argument information and to 
InfluxDB [documentation](https://docs.influxdata.com/influxdb/v1.3/query_language/spec/#queries)
for further query-related information.


### Other functionality

#### Authentication

TODO

#### Database selection

After the instantiation of the `AsyncInfluxDBClient` object, database can be switched 
by changing the `db` attribute:

```python
client = AsyncInfluxDBClient(db='db1')  # instatiate client
client.db = 'db2'  # switch database
```  

Beaware that differently from some NoSQL databases (such as MongoDB), InfluxDB requires that 
a databases is explicitly created (by using the 
[`CREATE DATABASE`](https://docs.influxdata.com/influxdb/v1.3/query_language/database_management/#create-database)
 query) before doing any operations on it.


#### Debugging

TODO


## Implementation

Since InfluxDB exposes all its functionality through an 
[HTTP API](https://docs.influxdata.com/influxdb/v1.3/tools/api/), 
`AsyncInfluxDBClient` tries to be nothing more than a thin and dry wrapper around that API.

The InfluxDB HTTP API exposes exactly three endpoints/functions: `ping`, `write` and `query`. 

`AsyncInfluxDBClient` merely wraps these three functions and provides some parsing functionality for generating 
line protocol data (when writing) and parsing JSON responses (when querying). 

Additionally, [partials](https://en.wikipedia.org/wiki/Partial_application) are used in 
order to provide convenient access to commonly used query patterns. 
See the [Convenience functions](#convenience-functions) section for details.


## Contributing

To contribute, fork the repository on GitHub, make your changes and submit a pull request.  
Aioinflux is not a mature project yet, so just simply raising issues is also greatly appreciated :)
