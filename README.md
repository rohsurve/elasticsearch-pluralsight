# Getting Started with ElasticSearch

> My course notes from learning ElasticSearch with [Pluralsight Course](https://app.pluralsight.com/library/courses/elasticsearch-for-dotnet-developers/table-of-contents)

## Concepts

Document is saved in _index_ (which is kind of like a Database in relational world).

Indexes are stored across multiple _shards_, which are logical ways of slicing the data into individual chunks so they can be stored across multiple servers.

Shards are stored in one or more servers, which are called _nodes_.

A collection of nodes is considered a _cluster_.

Clustering is what makes ES easy to scale horizontally - i.e. adding more servers to the cluster so the shares can be distributed to them, to help balance out the load.

## Installation

Needs at least jre7. Download install zip from site, unpack and run the startup bat or sh specified in instructions.

## Configuration

Open `config/elasticsearch.yml` in installation folder and set cluster and node name, for example:


```
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
cluster.name: esdemo
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
node.name: node01
#
```

Start elastic search, for example on Windows:

```shell
cd bin
elasticsearch.bat
```

Use Postman or CURL to verify that [http://localhost:9200](http://localhost:9200) returns a response.


Also install Kibana, Marvel and Sense follow installation and configuration instructions at:
* [Kibana](https://www.elastic.co/downloads/kibana)
* [Marvel](https://www.elastic.co/downloads/marvel)
* [Sense](https://github.com/elastic/sense/)

Note that Kibana is a trial license, not free.

Sense is a Kibana plugin.

Sense is at [http://localhost:5601/app/sense](http://localhost:5601/app/sense)

Given host of: http://localhost:9200, some sample queries are:

```
GET _search
{
  "query": {
    "match_all": {}
  }
}

GET /
```

Remainder of this course will use Sense plugin.

## Schemas (Mappings)

Example comparing to a relational database:

```sql
-- Create the database
CREATE DATABASE my_blog DEFALT CHARACTER SET latin1 COLLATE latin1_swedish_ci;

-- Create blog posts table
CREATE TABLE IF NOT EXISTS post (
  id bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  user_id int(10) NOT NULL,
  post_text varchar(255) DEFAULT NULL,
  post_date datetime NOT NULL,
  PRIMARY KEY (id)
) ENGINE=MyISAM DEFAULT CHARSET=UTF8 AUTO_INCREMENT=3;
```

To create an index with this as a schema (aka mapping) in ES, POST a mapping request as follows:

`POST http://localhost:9200/my_blog`

```json
{
  "mappings": {
    "post": {
      "properties": {
        "user_id": {
          "type": "integer"
        },
        "post_text": {
          "type": "string"
        },
        "post_date": {
          "type": "date"
        }
      }
    }
  }
}
```

`my_blog` will become the index name.

Instead of database "table", blog post is considered a "type", which has "properties".

Properties in ES are like columns in a relational database. Each has its own data type, similar to sql data types.

Notice the mapping above has no "id" field. This is because id's are treated specially in ES. By default, it will generate the id automatically.

To verify schema was created, check what indicies are present:

```
GET /_cat/indices
```

To verify mapping was created successfully:

```
GET /my_blog/_mapping
```

## Inserting documents

To insert a document:

```
POST /my_blog/post
{
  "post_date": "2015-08-20",
  "post_text": "This is a real blog post!",
  "user_id": 1
}
```

Insert another:
```
POST /my_blog/post
{
  "post_date": "2015-08-25",
  "post_text": "This is another real blog post!",
  "user_id": 2
}
```

## Basic search

To search ALL blog posts:

```
GET /my_blog/post/_search
```

Sample output for two docs inserted:

```json
{
  "took": 37,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 1,
    "hits": [
      {
        "_index": "my_blog",
        "_type": "post",
        "_id": "AVImVnsdIRq3mAlBy4Fh",
        "_score": 1,
        "_source": {
          "post_date": "2015-08-25",
          "post_text": "This is another real blog post!",
          "user_id": 2
        }
      },
      {
        "_index": "my_blog",
        "_type": "post",
        "_id": "AVImVNA-IRq3mAlBy4Dn",
        "_score": 1,
        "_source": {
          "post_date": "2015-08-20",
          "post_text": "This is a real blog post!",
          "user_id": 1
        }
      }
    ]
  }
}
```
To query by id (as generated by ES):

```
GET /my_blog/post/AVImVNA-IRq3mAlBy4Dn
```

If you don't want the ES auto generated id's and instead want to use your own id system, insert as follows, notice the "/1" at end of POST url means create this document using id=1:

```
POST /my_blog/post/1
{
  "post_date": "2015-08-26",
  "post_text": "This post should have an _id of 1",
  "user_id": 2
}
```

## Data Types

ES comes with 5 core data types out of the box:

* String - most common, similar to varchar in relational database. Use it to store any collection of characters. Many options and settings for flexibility.
* Boolean - pretty basic, either true or false
* Number - many options and settings, can be byte (8 bit integer), short (16 bit integer) Float (single precision 32 bit floating point), double. These correspond to core types in Java.
* Date - lots of formatting options. Stored as UTC, default format is "Date Optional Time", which means ES will append time string to value that is inserted. Can specify multiple formats at once.
* Binary - for example images, stored as base64 string, not indexed by default.

### Data Type Options

Most data types support additional options and settings to tweak their usage in an index, for example, to specify that entries must have date in YYYY-MM-DD format:

```json
{
  "mappings": {
    "post": {
      "properties": {
        "user_id": {
          "type": "integer"
        },
        "post_text": {
          "type": "string"
        },
        "post_date": {
          "type": "date",
          "format": "YYYY-MM-DD"
        }
      }
    }
  }
}
```

If a document is inserted with `post_date` not in specified format, ES will return a 400 error.

### Create Index with Settings

First delete previous index we created:

```
DELETE /my_blog/
```

To create an index specifying number of shards as a setting (good practice to always specify num shards in multi-node cluster):

`POST http://localhost:9200/my_blog`

```json
{
  "settings": {
    "index": {
      "number_of_shards": 5
    }
  },
  "mappings": {
    "post": {
      "properties": {
        "user_id": {
          "type": "integer"
        },
        "post_text": {
          "type": "string"
        },
        "post_date": {
          "type": "date",
          "format": "YYYY-MM-DD"
        }
      }
    }
  }
}
```
