---
layout: article
title: "Explore a JWT on the command line"
date: 2022-12-03 06:49:00 -0100
tags:
- JWT
- JSON
- Confluent license
summary: "How can we explore a JSON web token on the command line using jq"
---

JSON Web tokens can be explored and validated on [jwt.io](https://jwt.io/).
Alternatively, the following code can be used to display the content of a JWT:

```bash
cat license | jq -R 'split(".") | .[0],.[1] | @base64d | fromjson'
```

It can be further combined with the `todate` filter to display the validity dates:

```bash
cat license | jq -R 'split(".") | .[1] | @base64d | fromjson | .exp,.iat | todate'
```

N.B.: `iat` stands for *issued at*, `exp` stands for *expires at•.

This can be used, for example, to display and validate a Confluent license.
