---
title: "Writing null values and headers to Kafka topics"
date: 2022-08-27 08:06:00 -0000
categories: CLI Kafka
---

# Writing null values and headers to Kafka topics

Starting with Apache Kafka 3.2, it is possible to write headers and `null` values with the `kafka-console-producer`. This is useful for debugging purposes are to produce tombstone messages to compacted topics.

Writing `null` values is unlocked with the new property `null.marker`.

For example:
```shell
kafka-console-producer --bootstrap-server localhost:9092 --topic test --property null.marker=nil
>hello
>nil
>nil test
```
The second input will be parsed as `null`:
```shell
kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
hello
null
nil test
```
As we see in the last line, the `null` marker has to be entered verbatim for having an effect.

We can change the way `null` values are printed by the console-consumer by using the `null.literal` property:
```shell
kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning --property null.literal=NULL
hello
NULL
nil test
```
Note the different property names: `null.marker` for the producer, `null.literal` for the consumer.

`null` values are parsed in the key as well:
```shell
kafka-console-producer --bootstrap-server localhost:9092 --topic test --property null.marker=nil --property parse.key=true
>key1	value1
>key2	value2
>key1	nil
>nil	nil

kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning --property null.literal=NULL --property print.key=true
NULL	hello
NULL	NULL
NULL	nil test
key1	value1
key2	value2
key1	NULL
NULL	NULL
```
Here we used a tab-character to separate key and values. The separator can be customized with the `key.separator` property.

Support for parsing null values was added in [KIP-810](https://cwiki.apache.org/confluence/display/KAFKA/KIP-810%3A+Allow+producing+records+with+null+values+in+Kafka+Console+Producer).

## Parsing headers

[KIP-798](https://cwiki.apache.org/confluence/display/KAFKA/KIP-798%3A+Add+possibility+to+write+kafka+headers+in+Kafka+Console+Producer) added header-support to `kafka-console-producer` by enabling the `parse.headers` property:
```shell
kafka-console-producer --bootstrap-server localhost:9092 --topic header_test --property null.marker=nil --property parse.key=true --property parse.headers=true
>h1:h1-v	key1	value1
>h1:h1-v,h2:h2-v        key2	nil
>h1:h1-v,h2:nil	nil	value-nil
>h1:h1-v,h1:h1-v2	key3	value3
>nil	key-no-header	value-no-header
```
As in the example before, header, key, and values are separated by a tabulator (`\t`) by default. Header key/value pairs are separated by a comma (`,`) and a colon (`:`) is used to separate key and value of a single header. These settings can be overridden using the `headers.delimiter` and `key.separator` , `headers.separator`, and `headers.key.separator` properties.
The following example demonstrates the usage of these properties:
```shell
kafka-console-producer --bootstrap-server localhost:9092 --topic header_test --property null.marker=nil --property parse.key=true --property parse.headers=true --property key.separator=';' --property headers.separator=';' --property headers.delimiter='/' --property headers.key.separator='|'
>h1|hv-1;h2|hv-2/key;value
```

The last line demonstrates the use of the null-marker to specify an empty list of headers. Without specifying a null-marker, parsing will fail if no headers are specified.

```shell
kafka-console-consumer --bootstrap-server localhost:9092 --topic header_test --from-beginning --property null.literal=NULL --property print.key=true --property print.headers=true
h1:h1-v	key1	value1
h1:h1-v,h2:h2-v	key2	NULL
h1:h1-v,h2:NULL	NULL	value-nil
h1:h1-v,h1:h1-v2	key3	value3
NO_HEADERS	key-no-header	value-no-header
h1:hv-1,h2:hv-2	key	value
```
The `NO_HEADERS` literal to denote the absence of headers cannot be changed without writing a special formatter, as a quick look the [source code ](https://github.com/apache/kafka/blob/481fefb4f9ca0ecf83b72116977416d3a0472127/core/src/main/scala/kafka/tools/ConsoleConsumer.scala#L569)  shows. Also, the header-key-separator is hardcoded as a colon (`:`).


## Comparison with kcat

[`kcat`](https://github.com/edenhill/kcat) (formerly known as `kafkacat`) has supported producing headers and `null` values for quite some time using the following command line options:

- `-H` option allows to specify key-value pairs for header. These will be applied to all messages produced.
- `-K` option allows to specify key/value separator.
- `-Z` option will output empty strings as `null` values.

For example
```shell
kcat -P -b localhost:9092 -t kcat_test -K: -Z -H h1=v1 -H h2=v2
key1:value1
:value2
key3:
:
```
leads to the following:
```
h1:v1,h2:v2	key1	value1
h1:v1,h2:v2	NULL	value2
h1:v1,h2:v2	key3	NULL
h1:v1,h2:v2	NULL	NULL
```

As we see, `kcat` more concise but offers less options.
