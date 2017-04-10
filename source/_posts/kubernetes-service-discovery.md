---
title: Kubernetes 服务发现原理
date: 2017-03-16 11:10:23
tags:
  - Kubernetes
  - Docker
---

## 创建 server 之前，由 etcd 维护一个全局的注册表，分配 VIP

## Virtual IP

- Proxy-mode: userspace
- Proxy-mode: iptables
    This should be faster and more reliable than the userspace proxy. However, unlike the userspace proxier, the iptables proxier cannot automatically retry another Pod if the one it initially selects does not respond, so it depends on having working readiness probes.

## Why not use round-robin DNS?

- There is a long history of DNS libraries not respecting DNS TTLs and caching the results of name lookups.
- Many apps do DNS lookups once and cache the results.
- Even if apps and libraries did proper re-resolution, the load of every client re-resolving DNS over and over would be difficult to manage.

## 名词

- Deployment: The Deployment is responsible for creating and updating instances of your application.
- Pod: A Pod is a Kubernetes abstraction that represents a group of one or more application

<!-- more -->

## 未完待续...
