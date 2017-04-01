<img src="docs/_static/images/qg_logo.png" alt="Drawing" />

***

Query Graph is a framework/language, written in Python, for joining data 
from different database management systems - i.e. joins that can't 
typically be accomplished with a single query. For example, joining 
Postgres data and Mongo Db data. It also provides tools for easily 
converting non-tabular data (e.g. JSON) into tabular form.

The following databases are currently supported:

* Sqlite
* MySQL
* Postgres
* Mongo Db
* Elastic Search
* Apache Cassandra (untested)
* Maria Db (untested)
* InfluxDB (untested)
* MS Sql (untested)

## Main Features

* Join data from any number of different database types in a single query.
* Manipulate data using "manipulation sets", which are chained together
  statements very similar to those used in the `dplyr` package for R.
  ```
  mutate(new_col=log(old_col)) >>
  remove(some_column)
  ```
* Easily transform JSON-like data into relational form.
* Threading can optionally be used to run queries on different databases
  simultaneously, based the structure of the query graph.

## Getting Started

* To install Query Graph, see below.
* For a brief introduction to Query Graph, see here.
* For a more complete introduction, see here.
* For documentation, see here.

## Installation

To install Query Graph...

# Query Graph Language - Brief Introduction

Query Graph Language (QGL) is a simple, domain specific declarative 
language. The best way to give an idea of how it works is through
an example. In the example, we'll be joining data from two databases:
a Mongo Db database, and a Postgres database.

## The Mongo Db Database

The Mongo Db database contains a collection, `albums`, with data in the 
following form:

```
{
      "album": "Jagged Little Pill",
      "tags": [
        "canada",
        "pop rock",
        "post-grunge",
        "female"
      ],
      "data": {"record_label": "Maverick",
               "year": 1995}
}
```

## The Postgres Database

The Postgres database contains a table, `Albums`, that is structured in
the following manner:

| AlbumId | Title              | ArtistId |
|---------|--------------------|----------|
| ...     | ...                | ...      |
| 6       | Jagged Little Pill | 4        |
| ...     | ...                | ...      |


## The QGL Query

The QGL query used to join both sets of data is below.

```
CONNECT
    postgres_conn <- Postgres(db_name='', user='', password='', host='', port='')
    mongodb_conn <- Mongodb(host='', port='', db_name='', collection='')
RETRIEVE
    QUERY |
        {'tags': {'$in': {% album_tags -> list:str %}}};
    FIELDS album, tags, data
    USING mongodb_conn
    THEN |
        unpack(record_label=data['record_label']) >>
        unpack(year=data['year']) >>
        remove(data) >>
        flatten(tags) >>
        rename(tags=tag);
    AS mongo_node
    ---
    QUERY |
        SELECT *
        FROM "Album"
        WHERE "Title" IN {{ album -> list:str }};
    USING postgres_conn
    THEN |
        remove(Album);
    AS postgres_node
JOIN
    LEFT (postgres_node[Title] ==> mongo_node[album])
```

To execute the query in Python, the code is

```python
query_str = "..."

# Create the graph instance.
query_graph = QueryGraph(qgl_str=query_str)

# Execute the graph and get a dataframe in return.
df = query_graph.execute(album_tags=['canada', 'rock'])
```

The output is a Pandas dataframe:

| AlbumId | album | ArtistId | tag | record_label | year |
|---------|-------|----------|-----|--------------|------|
| ...     | ...   | ...      | ... | ...          | ...  |
| ...     | ...   | ...      | ... | ...          | ...  |
| ...     | ...   | ...      | ... | ...          | ...  |

Now we'll walk through the query and each step of its execution.

## Walkthrough

The basic structure of our example query is broken down in the diagram
below. The `CONNECT` block establishes connections to the databases
that will be queried upon execution. The `RETRIEVE` block is where
"query nodes" are defined, and each query node represents a query on 
a single database. The `JOIN` block describes how the data results of 
each query node are joined.

<hr>

![Parameter Diagram](docs/_static/images/ex_query_diagram.png)

<hr>

When our example query is executed, the first thing to happen is the
rendering of the "query template" belonging to the `mongo_node` query
node. A query template is simply a query written in the form appropriate
for whatever database type it will be executed on, augmented with
"template parameters". The `mongo_node` query template has a single 
"independent" parameter:

![Parameter Diagram](docs/_static/images/ex_mongo_param.png)

The parameter is "independent" because its value is defined before the
query is executed. The parameter "value expression" defines the value
assigned to the parameter. In our case, we defined

```python
album_tags=['canada', 'rock']
```

The "container type" and "render type" indicate that the value defined
above should be rendered as a list of strings. The rendered query for
`mongo_node` is:

```
{'tags': {'$in': ['canada', 'rock']}}
```

which is just an ordinary Mongo Db query (using their Python API). The
query is then executed using the `mongo_conn` connector, with only the
fields "album", "tags", and "data" selected.

| album              | tags                                           | data                                        |
|--------------------|------------------------------------------------|---------------------------------------------|
| Jagged Little Pill | `["canada","pop rock","post-grunge","female"]` | `{"record_label": "Maverick","year": 1995}` |
| ...                | ...                                            | ...                                         |
| ...                | ...                                            | ...                                         |