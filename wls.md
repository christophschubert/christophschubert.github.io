---
layout: default
title: Work work work
---

# Work life study log - January 2021

## January 1<sup>st</sup>, 2021
Bornreihe
- added SASL Plain config option to cp-testcontainers
- had to dig into the configuration of the CP Docker images
  Useful links and learnings:
  - Confluent Docs on how to [extend the images](https://docs.confluent.io/platform/current/installation/docker/development.html)
  - the source code for the `cub` and `dub` Python scripts to configure the images can be found here [on github](https://github.com/confluentinc/confluent-docker-utils/tree/master/confluent/docker_utils).

## January 2<sup>nd</sup>, 2021
Bremen
- Cleaned up test code for `cp-testcontainers`.
  Learnings:
  - When setting a custom wait strategy, `withStartupTimeout` needs to be called *after* `waitingFor`.
    Otherwise the default timeout of 60 seconds will still be used.
- Wrote tests for REST proxy container to use rest-assured.

## January 3<sup>rd</sup>, 2021
Bremen
- further cleaned up test code for `cp-testcontainers`
- added authorization option to SASL Plain Kafka test container
- published version v0.1.0 of [cp-testcontainers](https://github.com/christophschubert/cp-testcontainers)
- improved README of cp-testcontainers
- published v0.1.0 of [kafka-connect-java-client](https://github.com/christophschubert/kafka-connect-java-client)
