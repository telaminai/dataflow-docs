---
title: Integration
has_children: false
nav_order: 200
published: true
layout: default
---


# DataFlow integration
---

Integrating a DataFlow into an application is a critical part of delivering the reactive application. 

## Input
Supported input methods:

- Native DataFlow integration, application calls onEvent
- EventFeed re-uses prebuilt event input sources 
  - File event watcher
  - Kafka event feed
  - Debezium event feed
  - Custom written an event feed

## Output
Supported output methods:
- Sink registration
- Agents invoking external services
- Lookup nodes by id and query state
- Output sink re-uses prebuilt sink outputs
  - File sink
  - Kafka sink
  - JDBC

## Cache
A cache acts as both an input and output source to a DataFlow, examples include:
- File cache
- Redis
- Chronicle