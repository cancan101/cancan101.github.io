---
layout: post
title: "Running Apache Superset Against a GraphQL API"
date: 2022-03-09 17:39:58 -0500
comments: true
categories: business-intelligence graphql
description: I develop a SQLAlchemy Dialect and associated Python DB-API database driver that allows Apache Superset to query data from GraphQL API.
keywords: business intelligence, graphql, apache superset, sqlalchemy
---
In this blog post I walk through my journey of getting [Apache Superset](https://superset.apache.org/) to connect to an arbitrary [GraphQL API](https://graphql.org/). You can [view the code I wrote to accomplish this here](https://github.com/cancan101/graphql-db-api).
<!-- more -->

I wanted a way for Superset to pull data from a GraphQL API so that I could run this powerful and well used business intelligence (BI) tool on top of the breadth of data sources that can be exposed through GraphQL APIs. Unfortunately Superset wants to connect directly with the underlying database (from its landing page: "Superset can connect to [any SQL based datasource through SQLAlchemy](https://superset.apache.org/docs/databases/installing-database-drivers/)").

Generally this constraint isn't that restrictive as data engineering best practices are to set up a centralized data warehouse to which various production databases would be synchronized. The database used for the data warehouse (e.g. Amazon Redshift, Google BigQuery) can then be queried directly by Superset.

The alternative, "lazy" approach is to eschew setting up this secondary data storage system and instead point the BI tool directly at the production database (or, slightly better, at a [read replica of that database](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)). This alternative approach is the route often chosen by small, fast moving, early-stage product engineering teams that either lack the resources or know-how to engage in proper data engineering. Or the engineers just need something "quick and dirty" to empower the business users to explore and analyze the data themselves and get building ad-hoc reports off the engineering team's plate.

I find myself in in the latter scenario, where I want to quickly connect a BI tool to our systems without needing to set up a full data warehouse. There is a twist though. While our primary application database in Postgres (which is supported by Superset), we augment that data using various external / third-party APIs and expose all of this through a GraphQL API[^cache-etc]. In some cases there are data transformations and business logic in the [resolvers](https://graphql.org/learn/execution/#root-fields-resolvers) that I would like to avoid having to re-implement in the BI tool.

[^cache-etc]: We use [Redis](https://redis.io/) caching + [DataLoaders](https://github.com/graphql/dataloader) to mitigate the cost of accessing these APIs.

And with that I set out to connect Superset (my BI tool of choice) directly to my GraphQL API. It turns out someone else also had this idea and had kindly filed a [GitHub ticket back in 2018](https://github.com/apache/superset/issues/5389). The ticket unfortunately was closed with:<br>
&nbsp;&nbsp;a) It's possible to extend the built in connectors. "The preferred way is to go through SQLAlchemy when possible, otherwise you can look at what's under...".<br>
&nbsp;&nbsp;b) "All datasources ready for Superset consumption are single-table".<br>
While (b) could be an issue with GraphQL as GraphQL does allow arbitrary return shapes, if we confine ourselves to top level [`List`s](https://graphql.org/learn/schema/#lists-and-non-null) or [`Connection`s](https://relay.dev/graphql/connections.htm) and flatten related entities, the data looks "table-like". (a) boiled down to writing a [SQLAlchemy `Dialect`](https://docs.sqlalchemy.org/en/14/core/engines.html) and associated [Python DB-API database driver](https://www.python.org/dev/peps/pep-0249/). A quick skim of both the SQLAlchemy docs and the PEP got me a little worried at the what I might be attempting to bite off.

Before I gave up hope, I decided to take a quick look at [the other databases that Superset listed](https://superset.apache.org/docs/databases/installing-database-drivers/) to see if there was one I could perhaps base my approach off of. [Support for Google Sheets](https://superset.apache.org/docs/databases/google-sheets) caught my attention and upon digging further, I saw it was based on a library called [`shillelagh`](https://github.com/betodealmeida/shillelagh), whose docs say "Shillelagh allows you to easily query non-SQL resource" and "New adapters are relatively easy to implement. There's a step-by-step tutorial that explains how to create a new adapter to an API." _Intriguing_. On the surface this was exactly what I was looking for.

I took a look at the [step-by-step tutorial](https://shillelagh.readthedocs.io/en/latest/development.html), which very cleanly walks you through the process of developing a new `Adapter`. [From the docs](https://shillelagh.readthedocs.io/en/latest/adapters.html): "Adapters are plugins that make it possible for Shillelagh to query APIs and other non-SQL resources." Perfect!

The last important detail that I needed to figure out what how to map the arbitrary return shape from the API to something tabular. In particular I wanted the ability to leverage the graph nature of GraphQL and pull in related entities. However the graph may be infinitely deep, so I can't just crawl all related entities. I solved this problem by adding an "include" concept that specified what related entities should be loaded. this include is specified as a query parameter on the table name. In the case of [SWAPI](https://graphql.org/swapi-graphql), I might want to query the `allPeople` connection. This would then columns for all scalar fields of the `Nodes` returned. If I wanted to include the fields from the `species`, the table name would be `allPeople?include=species`. This would then add to the columns returned the fields on the linked `Species`. This path could be further traversed as `allPeople?include=species,species__homeworld`.

If you want to know more about the implementation of the `Dialect`, see the the [Implementation section](#Implementation), below.

Once the `graphql` `Dialect` is [registered with SQLAlchemy](https://superset.apache.org/docs/databases/docker-add-drivers), adding the GraphQL to Superset as a Database is as easy as creating a standard [Database Url](https://docs.sqlalchemy.org/en/14/core/engines.html#database-urls) where the `host` and `port` (optional) are the host and port of the API and the `database` portion is the path: `graphql://host(:port)/path/to/api`[^http].

{% img /images/post_images/2022-03-09-running-superset-against-graphql/database-config.png Database Config%}

[^http]: We do still need to specify the scheme (`http://` vs `https://`) to use when connecting to the GraphQL API. Options considered: <br>1. Two different Dialects (e.g.`graphql://` vs `graphqls://`), <br>2. Different Drivers (e.g. `graphql+http://` vs `graphql+https://`) or <br>3. Query parameter attached to the URL (e.g. `graphql://host:port/path?is_https=0`). <br>I went with (3) as most consistent with other URLs ([e.g. for Postgres: `?sslmode=require`](https://jdbc.postgresql.org/documentation/head/ssl-client.html)).


Once the Database is defined, the standard UI for defining Datasets can then be used, The list of "tables" is auto-populated through introspection of the GraphQL API[^dataset-name]:

{% img /images/post_images/2022-03-09-running-superset-against-graphql/dataset-add.png Adding a Dataset%}

[^dataset-name]: In order to set query params on the dataset name, I did have to toggle the `Virtual (SQL)` radio button, which then let me edit the name as free text:<br>{% img /images/post_images/2022-03-09-running-superset-against-graphql/dataset-name.png Setting Dataset Name %}

The columns names and their types can also be sync'ed from the Dataset:

{% img /images/post_images/2022-03-09-running-superset-against-graphql/dataset-columns.png Dataset Columns%}

Once the Database and the Dataset are defined, the standard Superset tools like charting can be used:

{% img /images/post_images/2022-03-09-running-superset-against-graphql/dataset-charting.png Dataset Charting%}

and exploring:

{% img /images/post_images/2022-03-09-running-superset-against-graphql/dataset-explore.png Dataset Exploring%}

Longer term there are a whole bunch of great benefits of using GraphQL which [this blog post](https://www.sspaeti.com/blog/analytics-api-with-graphql-the-next-level-of-data-engineering/) explores further.

# Appendix
## Implementation<a name="Implementation"></a>
The full source code can be [found on GitHub](https://github.com/cancan101/graphql-db-api). At some point soon, I will also publish to PyPI.

The majority of the logic is in the [`Adapter` class](https://github.com/cancan101/graphql-db-api/blob/main/graphqldb/adapter.py) and it roughly boils down to:<br>
1. Taking the table name (exposed on the GraphQL API as a field on `query`) along with any path traversal instructions and resolving the set of "columns" (flattened fields) and their types. This leverages [the introspection functionality of GraphQL APIs](https://graphql.org/learn/introspection/).<br>
2. Generating the needed GraphQL query and then mapping the results to a row-like structure.

I also [wrote my own subclass of `APSWDialect`](https://github.com/cancan101/graphql-db-api/blob/main/graphqldb/dialect.py), which while not documented in shillelagh's tutorial was [the approach taken for the `gsheets://` Dialect](https://github.com/betodealmeida/shillelagh/blob/a427de0b2d1ac27402d70b8a2ae69468f1f3dcad/src/shillelagh/backends/apsw/dialects/gsheets.py). This comes with the benefit of being able to expose the list of tables supported by the `Dialect`. This class leverages the GraphQL


## Development Experience
I love to early stage development work in a [Jupyter Notebook](https://jupyter.org/) with the [autoreload extension](https://ipython.readthedocs.io/en/stable/config/extensions/autoreload.html) (basically hot reloading for Python). However this presented a bit of a problem as both [SQLAlchemy](https://docs.sqlalchemy.org/en/14/core/connections.html#registering-new-dialects) and [shillelagh](https://shillelagh.readthedocs.io/en/latest/development.html#informing-shillelagh-of-our-class) expected their respective `Dialect`s and `Adapter`s to be registered as [entry points](https://packaging.python.org/en/latest/specifications/entry-points/).

[SQLAlchemy's docs provided a way to do this In-Process](https://docs.sqlalchemy.org/en/14/core/connections.html#registering-dialects-in-process):
```python
from sqlalchemy.dialects import registry

registry.register("graphql", "__main__", "APSWGraphQLDialect")
```

However shillelagh did not ([feature now requested](https://github.com/betodealmeida/shillelagh/issues/181)), but I was able to work around the issue:
```python
from pkg_resources import EntryPoint, Distribution
from shillelagh.backends.apsw import db

db.iter_entry_points = lambda x: [
    EntryPoint("graphql", "__main__", attrs=('GraphQLAdapter',), dist=Distribution())
]
```
