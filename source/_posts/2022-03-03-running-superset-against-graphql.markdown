---
layout: post
title: "Running Apache Superset Against a GraphQL API"
date: 2022-03-03 17:09:58 -0500
comments: true
categories: business intelligence, graphql
description: I develop a SQLAlchemy Dialect and associated Python DB-API database driver that allows Apache Superset to query data from GraphQL API.
keywords: business intelligence, graphql, apache superset, sqlalchemy
---
In this blog post I walk through my journey of getting [Apache Superset](https://superset.apache.org/) to connect to an arbitrary [GraphQL API](https://graphql.org/). You can see the library I built to accomplish that: [github.com/cancan101/graphql-db-api](https://github.com/cancan101/graphql-db-api).
<!-- more -->

I wanted a way to have Apache Superset pull data from a GraphQL API. With that I could then use a powerful and well used business intelligence along with the breadth data sources that can be exposed through GraphQL APIs. Unfortunately Superset wants to connect directly with the underlying database ("Superset can connect to any SQL based datasource through SQLAlchemy"). Generally this makes sense and is consistent with best practices of setting up a centralized data warehouse to which various production databases would be synchronized. The alternative, "lazy" approach is to eschew setting up this secondary data storage system and instead point the BI tool at the production database (or slightly better at a read replica of that database). The latter approach of using the production database or a read replica of it is the route often chosen by small, fast moving, often nascent teams that either lack the resources or know-how to engage in proper data engineering.

And with that I set out with objective of connecting Superset directly to a GraphQL API. It turns out some one else had this idea and had filed a [GitHub ticket back in 2018](https://github.com/apache/superset/issues/5389). The ticket was closed with a) it's possible to extend the built in connectors and b) "All datasources ready for Superset consumption are single-table". While the latter point could be an issue with GraphQL as GraphQL does allow arbitrary return shapes, if we confine ourselves to top level arrays or Connections and flatten related entities, the data looks "table-like". The former constraint boiled down to writing a [SQLAlchemy `Dialect`](https://docs.sqlalchemy.org/en/14/core/engines.html) and associated [Python DB-API database driver](https://www.python.org/dev/peps/pep-0249/). A quick skim of both the SQLAlchemy docs and the PEP got me a little worried at the what I might be attempting to bite off. Before I gave up hope, I decided to take a quick look at [the other databases that Superset listed](https://superset.apache.org/docs/databases/installing-database-drivers/) to see if there was one I could perhaps base my approach off off. [Support for Google Sheets](https://superset.apache.org/docs/databases/google-sheets) caught my attention and upon digging further, I saw it was based on a library called [`shillelagh`](https://github.com/betodealmeida/shillelagh), whose docs say "Shillelagh allows you to easily query non-SQL resource" and "New adapters are relatively easy to implement. There's a step-by-step tutorial that explains how to create a new adapter to an API." Intriguing. On the surface this was exactly what I was looking for.

I took a look at the [step-by-step tutorial](https://shillelagh.readthedocs.io/en/latest/development.html), which very cleanly walk you through the process of developing a new adapter. [From the docs](https://shillelagh.readthedocs.io/en/latest/adapters.html): "Adapters are plugins that make it possible for Shillelagh to query APIs and other non-SQL resources."

{% img /images/post_images/2022-03-03-running-superset-against-graphql/dataset-columns.png Dataset Columns%}

{% img /images/post_images/2022-03-03-running-superset-against-graphql/dataset-charting.png Dataset Charting%}

# Appendix
https://shillelagh.readthedocs.io/en/latest/install.html#installation
https://rogerbinns.github.io/apsw/download.html#easy-install-pip-pypi
https://github.com/rogerbinns/apsw/issues/310

XXXXXXXXX


https://precog.com/integrations/github-graphql-to-apache-superset/
https://www.sspaeti.com/blog/analytics-api-with-graphql-the-next-level-of-data-engineering/
Registering new SQLALchemy Dialect: https://docs.sqlalchemy.org/en/14/core/connections.html#registering-new-dialects

no dedicated data engineering team or resources
wanted to empower business users to access / explore data (traditional BI use case)
our primary internal application is based on GraphQL.
backed by primary a postgres DB _but_ also various APIs that are used to augment the data. we use redis caching + dataloaders to mitigate the cost of accessing. In some cases there is data transformation / business logic in the resolvers that would like to avoid being duplicated.

longer term as we add more microssevices, whether backed by same databses / different databases, other saas tools / APIs we plan to use gql to stitch all that to gether to present a unified api to our applications. also is convnenient for data consumtions

authorization control
