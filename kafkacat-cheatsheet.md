---
layout: default
title: kafkacat cheatsheet
---

# kafkacat cheatsheet

* `-C` consume mode
* `-b` bootstrap servers

* `-c1` stop after one message
* `-t <topicname>` specify topic 

```
kafkacat -b localhost:9092 -C -c1 -t my-topic | hexdump -C
```
