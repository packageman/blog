---
title: Rancher DNS 实现原理总结
date: 2017-03-06 09:03:14
tags:
---

有关 Rancher dns 的用途[官网](https://docs.rancher.com/rancher/v1.5/en/cattle/internal-dns-service/)已经说的很明白了，我就不再画蛇添足了 :)

特别说明，本次试验的 Rancher 版本为 [Rancher 1.5](https://github.com/rancher/rancher/tree/v1.5.0), 其中 [Rancher dns 为 v0.14.1](https://github.com/rancher/rancher-dns/tree/v0.14.1), [Rancher metadata 为 v0.8.11](https://github.com/rancher/rancher-metadata/tree/v0.8.11)。

## Rancher dns 创建

Rancher dns 是在创建完 Rancher server 后添加 host 到集群中时作为系统服务自动创建的， 与此同时还会创建 Rancher metadata, Rancher scheduler 等一些其它的系统服务。

提到 Rancher dns 必须要提的另一个系统服务就是 Rancher metadata。因为：

- Rancher dns 是作为 Rancher metadata 的 sidekick service 存在的并且 Rancher dns 共用了 Rancher metadata 的网络。
- Rancher dns 是依赖 Rancher metadata 中保存的数据更新 dns 记录的

下图是添加 host 后自动生成的 Rancher metadata 和 dns 配置文件
![Rancher-compose-of-metadata-and-dns](/images/rancher-dns/metadata-dns-rancher-compose.png)

<!-- more -->

## Rancher dns 网络

刚才已经提到 Rancher dns 共用 Rancher metadata 的网络，那就从 Rancher metadata 的网络开始说起，从上图中的配置中可以看到 Rancher metadata 使用的网络是 `network_mode: bridge`, 即 Docker 默认的 `docker0` 网络。那么 Rancher metadata 的 ip 地址应该在 172.16.11.0/24 网络中(如果你没有更改 docker 默认 bridge 网络配置的话)。我们进入容器验证一下：

```
371: eth0@if372: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:6d:cf:df:f6:70 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.16.11.3/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet 169.254.169.250/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::6d:cfff:fedf:f670/64 scope link
       valid_lft forever preferred_lft forever
```

和我们期望的一样， Rancher metadata 的容器中存在一个 172.16.11.3 的 ip 地址

但是 169.254.169.250 这个 ip 地址是哪来的呢，又是干什么用的呢？这个我们还要回过头去看一下上面的配置文件。可以看到 metadata 的启动命令是 `start.sh rancher-metadata -subscribe`。秘密就藏在这个 `start.sh` 文件中，找到 [`start.sh`](https://github.com/rancher/rancher-metadata/blob/v0.8.11/package/start.sh) 源码看一下：

```bash
#!/bin/bash

ip addr add 169.254.169.250/32 dev eth0
exec "$@"
```

原来如此，Rancher 在启动 metadata 服务之前添加了 ip 169.254.169.250。

这里要注意的一个事情是 Rancher metadata 使用的是 bridge 网络，这个网络是不能跨 host 通信的。只能被所在 host 中的容器所访问。那其它 host 上的服务的变化又是如何更新本台 host 上的 metadata 的呢？还要回到 Rancher metadata 的启动命令, 发现有个 `-subscribe` 参数，根据 [Rancher 文档](https://github.com/rancher/rancher-metadata/tree/v0.8.11)上的说明 `-subscribe` 表示订阅 Rancher events，当有服务更新后 Racnher 会发送 event, 所有订阅了这个 event 的 Rancher metadata 服务都会更新自己的数据。

还应该注意的是 Rancher metadata 和 dns 配置中都有一个 `io.rancher.scheduler.global: 'true'` 的标签，这个标签表明 metadata 和 dns 服务要在每台 host 上都部署。所以，根据我们上面的分析它们之间是相互隔离的。

## Rancher dns 更新

前面已经提到当有服务更新时会触发 Racher event, Rancher metadata 接受到 event 后会更新自己的 metadata 并通知 Rancher dns metadata 有新版本。参考以下代码片断：

```go
// https://github.com/rancher/rancher-metadata/blob/v0.8.11/main.go#L149
if ctx.Bool("subscribe") {
  logrus.Info("Subscribing to events")
  s := NewSubscriber(os.Getenv("CATTLE_URL"),
    os.Getenv("CATTLE_ACCESS_KEY"),
    os.Getenv("CATTLE_SECRET_KEY"),
    ctx.String("answers"),
    sc.SetAnswers) // 订阅 Rancher event 的事件回调
  if err := s.Subscribe(); err != nil {
    logrus.Fatal("Failed to subscribe", err)
  }
}

// https://github.com/rancher/rancher-metadata/blob/v0.8.11/main.go#L207
func (sc *ServerConfig) SetAnswers(versions Versions) {
  sc.Lock()
  defer sc.Unlock()
  sc.versions = versions
  sc.versionCond.Broadcast() // 通知 Rancher dns metadata 版本变化
}
```

图为 Rancher metadata 服务处理 Rancher event 的 log
![metadata-processing-event](/images/rancher-dns/metadata-process-event.jpg)

Rancher dns 是采用长轮询实现的实时更新。Rancher dns 循环向 Rancher metadata 服务发送请求检查 metadata 是否有变化，超时时间是 5s, 如果有版本变化，Rancher dns 会下载最新的 metadata 并更新自己的 dns 记录。否则服务器将等待直到超时返回。相关代码如下：

```go
// https://github.com/rancher/go-rancher-metadata/blob/master/metadata/change.go#L15
func (m *client) onChangeFromVersionWithError(version string, intervalSeconds int, do func(string)) error {
  for {
    newVersion, err := m.waitVersion(intervalSeconds, version)
    if err != nil {
      return err
    } else if version == newVersion {
      logrus.Debug("No changes in metadata version")
    } else {
      logrus.Debugf("Metadata Version has been changed. Old version: %s. New version: %s.", version, newVersion)
      version = newVersion
      do(newVersion) // 更新 dns 记录
    }
  }

  return nil
}

// https://github.com/rancher/go-rancher-metadata/blob/master/metadata/change.go#L51
func (m *client) waitVersion(maxWait int, version string) (string, error) {
  for {
    resp, err := m.SendRequest(fmt.Sprintf("/version?wait=true&value=%s&maxWait=%d", version, maxWait))
    if err != nil {
      t, ok := err.(timeout)
      if ok && t.Timeout() {
        continue
      }
      return "", err
    }
    err = json.Unmarshal(resp, &version)
    return version, err
  }
}
```

图为 Rancher dns 服务更新 dns 记录的 log
![dns-update-answers](/images/rancher-dns/dns-update-answers.jpg)

## Rancher dns 查询

Rancher dns 只有在使用 `managed` 网络的容器中才可以使用。在使用 `managed` 网络创建的容器会自动将 `/etc/resolv.conf` 中的 nameserver 设置为 169.254.169.250。这个地址就是我们之前提到的在创建 Rancher metadata 服务时生成的 ip 地址。

- Rancher dns 会首先查询缓存，命中则直接返回
- 如果没有命中则查询 Rancher dns 记录中是否有符合条件的。有则将记录加入到缓存中并返回
- 如果没有找到匹配的，则使用 docker 配置的 dns 地址进行递归查询。如果查询到加入到缓存中并返回

以下是相关代码：

```go
// https://github.com/rancher/rancher-dns/blob/v0.14.1/main.go#L211
func route(w dns.ResponseWriter, req *dns.Msg) {
  ...
  clientIp, _, _ := net.SplitHostPort(w.RemoteAddr().String())
  ...

  // 查询缓存
  if msg := clientSpecificCacheHit(clientIp, req); msg != nil {
    Respond(w, req, msg)
    log.WithFields(log.Fields{"client": clientIp, "type": rrString, "question": fqdn}).Debug("Sent client-specific cached response")
    return
  }

  if msg := globalCacheHit(req); msg != nil {
    Respond(w, req, msg)
    log.WithFields(log.Fields{"client": clientIp, "type": rrString, "question": fqdn}).Debug("Sent globally cached response")
    return
  }

  // 向 Rancher dns 记录中查找配置项
  // A records may return CNAME answer(s) plus A answer(s)
  if question.Qtype == dns.TypeA {
    found, ok := answers.Addresses(clientIp, fqdn, nil, 1)
    if ok && len(found) > 0 {
      log.WithFields(log.Fields{"client": clientIp, "type": rrString, "question": fqdn, "answers": len(found)}).Debug("Answered locally")
      m.Answer = found
      // 添加到缓存中
      addToClientSpecificCache(clientIp, req, m)
      Respond(w, req, m)
      return
    }
  } else {
    ...
  }

  // 根据 docker 配置的 dns 地址递归查询
  // Phone a friend - Forward original query
  msg, err := ResolveTryAll(req, answers.Recursers(clientIp))
  if err == nil && msg != nil {
    msg.Compress = true
    msg.Id = req.Id

    // 添加到全局缓存中
    addToGlobalCache(req, msg)

    Respond(w, req, msg)
    log.WithFields(log.Fields{"client": clientIp, "type": rrString, "question": fqdn}).Debug("Sent recursive response")
    return
  }

  // I give up
  log.WithFields(log.Fields{"client": clientIp, "type": rrString, "question": fqdn}).Info("No answer found")
  dns.HandleFailed(w, req)
}
```

## Rancher dns 缓存

Rancher dns 有两种缓存。一种是 `clientSpecificCaches` 另一种是 `globalCache`。两种缓存的过期时间默认都是 600s, 可以在启动 Rancher dns 服务的时候加入 `-ttl` 参数指定缓存过期时间。其中 `clientSpecificCaches` 是一个 Map 结构，key 是客户端的 ip。以下是 Rancher dns 代码中的相关定义

```go
// https://github.com/rancher/rancher-dns/blob/v0.14.1/main.go#L43
var (
  ...
  globalCache               *cache.Cache
  clientSpecificCaches      map[string]*cache.Cache
  ...
)
```

当 Rancher dns 监听到 metadata 改变更新 dns 记录的时候会清空 `clientSpecificCaches` 缓存，而 `globalCache` 则一直不会清空(直到缓存超时)。以下是相关代码：

```go
// https://github.com/rancher/rancher-dns/blob/v0.14.1/main.go#L123
func loadAnswersFromMeta(name string) {
  newAnswers, err := configGenerator.GenerateAnswers()
  ...

  if reflect.DeepEqual(newAnswers, answers) {
    log.Debug("No changes in dns data")
    return
  }

  // 清空缓存
  clearClientSpecificCaches()

  answers = newAnswers
  // write to file (debugging purposes)
  b, err := json.Marshal(answers)
  if err != nil {
    log.Errorf("Failed to marshall answers: %v", err)
  }
  err = ioutil.WriteFile(*answersFile, b, 0644)
  if err != nil {
    log.Errorf("Failed to write answers to file: %v", err)
  }
  log.Infof("Reloaded answers")
}
```

