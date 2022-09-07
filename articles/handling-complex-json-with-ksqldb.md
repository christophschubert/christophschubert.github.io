---
layout: article
title: "Handling complex JSON documents with ksqlDB"
date: 2022-09-07 06:39:00 -0100
tags:
- ksqlDB
- JSON
summary: "How can we work with nested JSON documents in ksqlDB?"
---

In this article, we will demonstrate how we can deal with nested JSON documents in [ksqlDB](https://ksqldb.io/).
When using `JSON` as data-format, we will use a *schema on read*  approach.
That means, we will have to specify the fields we want to extract from the JSON documents to form our stream when we specify the stream.

In order to demo this, we will use data from the Twitter stream API produced with the [Twitter Kafka producer](https://github.com/christophschubert/twitter-producer).
This data has the general shape, where we skipped all fields which we will not consider:

```json
{
  "created_at": "Tue Jul 12 04:39:54 +0000 2022",
  "text": "...",
  // skipped fields
  "user": {
    "id": 0,
    "name": "...",
    "screen_name": "...",
    "followers_count": 0,
    "created_at": "Wed Aug 04 04:36:48 +0000 2021"
    // skipped fields
  },

  // skipped fields
  "entities": {
    "hashtags": [],
    // skipped fields
    "user_mentions": [
      {
        "name": "...",
        "id": 0
        // skipped fields
      }
    ],
    "symbols": []
  },
  "lang": "...",
}
```
In order to read in this data, we create a ksqlDB stream with fields matching the keys in the JSON document, using `array` and `struct` to model sub-documents:

```  
create stream feed_raw (  
  text varchar,  
  created_at varchar,  
  user struct<
    id bigint,
    name varchar,   
    screen_name varchar,   
    followers_count integer,   
    created_at varchar  
  >,  
  entities struct<
    hashtags array<
      struct<
        text varchar,
        indices array<integer>  
      >
    >,
    user_mentions array<  
        struct<
          name varchar,
          id bigint  
        >
    >
  >,
  lang varchar)
with (kafka_topic='feed_raw', value_format='json');  
```
After creating the streams, we can query data from it as follows:
```
show streams;  

select text from feed_raw emit changes;  

select * from  feed_raw where lang= 'en' emit changes;  

select text, lang from feed_raw where lang='de' emit changes;  
```
We can access sub-documents using the `->` operator:
```
create stream feed_restructured as select  
  text,  
  user,  
  created_at,  
  entities->hashtags as hashtags,  
  entities->user_mentions as user_mentions,  
  lang as language
from feed_raw emit changes;  

# query new stream:  
select text, hashtags from feed_restructured emit changes;  

select text, hashtags, ARRAY_LENGTH(hashtags) as num_hashtags from feed_restructured emit changes;  
```
In order to access individual elements of the hashtags array, we can make use of the `explode` [table function](https://docs.ksqldb.io/en/latest/developer-guide/ksqldb-reference/table-functions/).
Table functions work similarly to `flatMap` known from many functional programming languages:
```
create stream tags as select
  user->name,
  explode(hashtags)->text as tag,
  language
from feed_restructured emit changes;  
```

## User-defined types

We can structure the code slightly better by first defining types for the nested documents:
```sql  
create type user_mention as struct<name varchar, id bigint>;  

create type hash_tag as struct<text varchar,  indices array<integer>>;  

create type entity as struct<
  hashtags array<hash_tag>,
  user_mentions array<user_mention>
>;  

create type user as struct<
  id bigint,   
  name varchar,   
  screen_name varchar,   
  followers_count integer,   
  created_at varchar  
>;  
```
These types can be referenced when creating a stream:
```
create stream feed_raw (  
  text varchar,  
  created_at varchar,  
  user user,  
  entities entity,  
  lang varchar
) with (kafka_topic='feed_raw', value_format='json');  

select
  user->name,
  user->followers_count,
  entities->hashtags,
  text
from feed_raw emit changes;  
```

## Missing fields
When we try to add a field which is not present in some or all of the messages, `null` values will be filled:  
```sql  
create stream feed_raw_non_existing_fields (  
  text varchar,  
  user user,
  unknown_text varchar,  
  unknown_int integer,  
  unknown_struct struct<a varchar, b integer>
) with (kafka_topic='feed_raw', value_format='json');  

select
  user->name,
  unknown_text,
  unknown_int,
  unknown_struct,
  text
from feed_raw_non_existing_fields emit changes;  
```  

## Building up complex data
Now that we have seen how we can de-construct a struct using the `->` syntax, how can we build up a struct?
This can be done using the `:=` operator:

```  
select
  struct(language := lang, user := user->name) as meta,  
  text as content  
from feed_raw emit changes;  
```
<!-- Think about adding types here? -->
## Output data
To write data back to a Kafka topic, use `create stream as`:
```
create stream summarized_json with (format='JSON') as  
select  
struct(language := lang, user := user->name) as meta,  
text as content  
from feed_raw emit changes;  
```
Observe that a topic with the name in the stream in upper-case and a `null` key was created.   

To add keys, one can add a key modifier:  
```sql  
create stream summarized_avro with (
  format='Avro',
  kafka_topic='summarized',
  partitions=6
) as select  
  user->name key,  
  struct(language := lang, user := user->name) as meta,  
  text as content  
from feed_raw partition by user->name emit changes;  
```  
In this case the output does not contain the key-field `name` in the value of the Kafka messages.  
<!-- // insert screnshot   -->

## Controlling field name cases
In order to control to case of the key-names, once can use back ticks:  
```sql  
create stream summarized_json_lowercase with (format='JSON') as  select  
  struct(`language` := lang, `user` := user->name)  as `meta`,  
  text as `content`  
from feed_raw emit changes;  
```  
This also works when using Avro, ProtoBuf, or JSON-Schema as serialization format:
```sql
create stream summarized_json_lowercase_sr with (format='JSON_SR') as select  
  struct(`language` := lang, `user` := user->name)  as `meta`,  
  text as `content`  
from feed_raw emit changes;
```

<!--
Bonus queries which don'st fit into the flow
```
  create table language_counts as select language, count(*) as count from  FEED_RESTRUCTURED group by  LANGUAGE emit changes;  
describe language_counts;  

select * from language_counts emit changes;  

select lcase(tag), count(*) from tags window tumbling (size 20 seconds) group by lcase(tag) emit changes;  

  create table language_descriptions (rowkey varchar  key, name varchar) with (kafka_topic='lang_table', value_format='json', partitions=1);  

insert into language_descriptions values ('de', 'Deutsch');  
insert into language_descriptions values ('en', 'Englisch');  
insert into language_descriptions values ('ru', 'Russisch');  


create table language_count_enriched with (kafka_topic='lce', value_format='json') as select d.name, c.count from LANGUAGE_COUNTS c inner join  LANGUAGE_DESCRIPTIONS d on c.rowkey = d.rowkey   emit changes;  

```   -->
