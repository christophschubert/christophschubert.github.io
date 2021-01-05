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
- published [v0.1.0](https://github.com/christophschubert/kafka-connect-java-client/releases/tag/v0.1.0) of [kafka-connect-java-client](https://github.com/christophschubert/kafka-connect-java-client)

## January 4<sup>th</sup>, 2021
Bremen
- hacked on `cp-testcontainers`
  - added RBAC enabled container for `cp-server` images
  - wrote decorator to give schema registry access to RBAC enabled cluster
  - cleaned up tests

## January 5<sup>th</sup>, 2021
Bremen
- hacked on `cp-testcontainers`: added RBAC configuration for connect

Learnings:
1. We can use the following environment variables to configure the log-level of different loggers:
  ```
  CONNECT_LOG4J_LOGGERS: org.eclipse.jetty=DEBUG,org.reflections=ERROR,org.apache.kafka.connect=DEBUG
  ```
  sets the jetty logger to DEBUG, etc.
  See [Confluent docs](https://docs.confluent.io/platform/current/connect/logging.html) for a description of the different loggers available in Connect.
1. When configuring Connect with RBAC, we need to add the following REST extension property:
  ```
  rest.extension.classes=io.confluent.connect.security.ConnectSecurityExtension
  ```
  When RBAC is enabled by secret registry is *not* enabled, one *must not* add the `io.confluent.connect.secretregistry.ConnectSecretRegistryExtension`.
  If it is added, the Connect REST API will return 404 for all requests.
  Of course, the extension has to be enabled for secret registry.
1. Setting the `confluent.balancer.topic.replication.factor` to 1 will also set the `_confluent-telemetry-metrics` topic RF to 1.
1. The `configure` script for `cp-server` container will not only transfer environment variables starting with `KAFKA_` into property values, but also those starting with `CONFLUENT` (I should double check that!)
