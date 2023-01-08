title: kube-apiserver 持续告警 KubeAPIErrorsHigh 调查
date: 2023-1-9 02:33
tags:
- KubeAPIErrorsHigh
- Too large resource version
- Kubernetes
- kube-apiserver
- 504
categories: Kubernetes

---
`kube-apiserver` 莫名其妙报 KubeAPIErrorsHigh 告警，CPU/Memory/Disk 都是好的。结果一路查下去似乎进了不知道有多深的兔子洞，坑深到这个问题从 2018 年到至今都没有完成彻底的修复。爬了很多楼，过程中学到了一些东西，这个文章全当做一个记录，假如有其他倒霉蛋看到这边也碰到了这个问题，希望这个文章能帮助到你。

<!-- more -->

# 现象

事情的起因都是千篇一律的：客户环境驻场反馈用户的 Prometheus 告警规则内的 KubeAPIErrorsHigh 项目持续告警了一个月，简单查看该告警对应的 PromSQL 如下：

```
    - alert: KubeAPIErrorsHigh
      annotations:
        message: API server is returning errors for {{`{{ $value }}`}}% of requests.
        runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeapierrorshigh
      expr: |-
        sum(rate(apiserver_request_count{job="apiserver",code=~"^(?:5..)$"}[5m])) without(instance, pod)
          /
        sum(rate(apiserver_request_count{job="apiserver"}[5m])) without(instance, pod) * 100 > 10
```

大概意思是 `kube-apiserver` 的 API 5分钟的请求中，返回码是 `5xx`  的占总请求数量的 10% 以上了。通过直接在 Prometheus 内查询对应的 Metric 指标，锁定有 2 台 `kube-apiserver` 对于特定的几个资源类型总是返回 504 超时。立刻翻看对应的 `kube-apiserver` 的日志，含有大量慢请求的 Trace 日志，并在 3s 后超时。


# 调查弯路

通常情况下，`kube-apiserver` 超时都是因为无良客户端无止境的昂贵请求把 `kube-apiserver` 打死。 最典型的是在执行 LIST 请求的时候，由于没有带有 `resourceVersion` 参数导致 `kube-apiserver` 内的 `watch-cache` 无法提供缓存，请求穿透缓存打在了 `etcd3` 的存储实现上。 由于改过程是一个 `quorum read`，加之返回数据量较大的时候（通常在大规模集群内，如上万个 Pod 规模情况下，LIST pod 的响应能够达到 500-800MB），很容易将 `etcd` 和 `kube-apiserver` 的 CPU和内存负载打到极高，进而产生大量的超时。这种超时往往是雪崩式、灾难级的，因为这很可能导致如 `lease` 之类的关键请求超时，继而触发更多的 Controller 换主/发生 relist 的情况，而这部分 LIST 请求无疑会更加加剧 `kube-apiserver` 的负载，最终导致控制面挂掉且无法恢复。

这波 504 超时想当然的按照上面的怀疑开始调查：
1. 观察 `kube-apiserver` 的负载：50% CPU 以下 + 1.3G RSS 内存，根本负载情况完全正常；
2. 观察 `etcd` 的负载：15% 的 CPU + 10G RSS 内存，CPU 很低内存偏高，不过整体上也没啥问题；
3. 观察 `etcd` 日志以及对应数据盘负载：日志内只有很久之前出现过两次慢 IO，磁盘整体上使用率很低，也看不出啥问题；

至此陷入了僵局，看起来不像是负载问题导致的。加上现场环境比较特殊，是国产 CPU + 混部环境的组合。因此在排除对硬件性能的不信任、以及复杂的 cGroup 绑核策略可能的怀疑上，浪费了较多的时间。

后面进行了一些无目的复现和尝试：
1. 对 audit log 内 504 的请求通过 `kubectl` 进行尝试请求，反向返回一切正常；
2. 分析 audit log，进行数据统计。发现问题只集中在特定的 2 台 `kube-apiserver` 上，并且错误并不随机：对于给定的 `kube-apiserver`，报错的客户端就那几个特定的 Pod。

结合以上两点，很容易怀疑到长链接的网络链路上。由于现场只能通过驻场手动执行 + 拍照来交互，对 `tcpdump` 抓包和对 `kube-apiserver` 的 pprof 分析显得就不太现实，最终只能通过尝试性重启 `kube-apiserver` 来观察是否能短期解决问题。事实上，在重启后的确不在报 504 异常，问题得到了临时性的解决。

# 原因定位

次日在开发环境，同样的版本遇到了相同问题。由于可以直接 SSH 到机器上看，因此过程就变得高效很多，这次发现了更多的线索：
1. 客户端和现场中虽然不是一个 Pod，但是从服务端的错误码上看，都是 504。在该 Pod 的日志内，观察到了大量的错误日志：`Timeout: Too large resource version: 2564, current: 2459`；
2. 之前 `kubectl` 的重放请求不够精确，只是简单的执行了 `kubectl get foo`，而没有和实际的请求完全一致。将 audit log 中的 `requestURI` 内的请求完全复制过来，通过 `kubectl get --raw /foo?resourceVersion=2564` 进行请求后，获得到了和`(1)` 中一致的超时报错；

复现了就好办了，借助 Google 的大力协助发现 Redhat 上很早就发现了类似的问题[Bug #1877346](https://bugzilla.redhat.com/show_bug.cgi?id=1877346)。通过对爬楼 [Issue # 91073](https://github.com/kubernetes/kubernetes/issues/91073) 以及 Review [PR #92537](https://github.com/kubernetes/kubernetes/pull/92537) 的代码，问题原因就很明确了。在 `client-go` 的 `reflactor` 实现中，如果因为某些原因 `Watch` 过程中断需要发生 relist 的时候，代码内只判断了是否是请求的 RV(resourceVersion) 太旧的异常，而没有判断太新。

这里要能明白这两个异常，需要补充一下 RV 相关的背景知识。正常情况下，存储在 `etcd` 内的数据都会带有一个版本号，这个版本号是单调递增的，且每次修改都会导致对应 KV 的版本改变。借助这个机制，`kube-apiserver` 就可以提供一个线性一致性的缓存机制：客户端请求某个资源的时候，不希望看到比上次看到更老的版本，即发生回到过去的情况。那么他只需要在请求的时候将版本号带上，`kube-apiserver` 就可以很简单判断，从而保证只返回比这个版本号更新的数据。此时，就会有两种异常情况：
1. 客户端请求的版本是一个很旧的版本，这个版本已经被淘汰掉不在缓存之中了。那么 `kube-apiserver `将会返回 `410 Gone` 这个异常，并希望 Client 能够重新执行一次全新的 List-Watch；
2. 客户端请求的版本是一个很新的版本，这个版本还没有被同步进缓存之中。那么 `kube-apiserver` 将会等待一段时间（3秒），希望在这段时间内可以等到 etcd 那边实际的事件被同步进缓存。如果等到了，自然是正常返回，而没有等到那就会返回我们遇到的那个 `Timeout: Too large resource version` 异常。

由于之前的版本内，只处理了太旧，没有处理太新，那么其实把这两种情况看作一样的情况去处理是不是就行了呢？ 实际上在 PR #92537 中，就是这么修复的。并且在 Issue #91073 中提到，这个 `client-go` 只影响到了 0.18.0 - 0.18.6 的客户端，0.17.x 不受到影响是因为引入 Bug 的 PR 被碰巧 Revert 掉了。所以理论上，只要把相关 Pod 的 `client-go` 升级到 0.18.6 之后的小版本就可以了。


# 进坑

按照上面的调查结果，只要升级 `client-go` 就行了。反过来说出问题的 Pod 应该都是在 0.18.0 - 0.18.6 之中的版本才对。正常如果故事结束，下面的事情就是给他们开一个 Bug，截图说明 `go.mod` 里这个地方要改下，然后在找人扫描下其他的代码仓库是否也有相关问题，有问题都开一遍 Bug 就结束了，但是这个问题还远远没有结束：
1. 问题相关的 Pod 的源码中，`client-go` 版本分别是 `0.22` 和 `0.20`，而这个问题在 `0.18` 就应该已经被修复了，WTF？
2. 我们的 K8S 版本是 1.16，1.18 之前的 `client-go` 没问题是因为一个 [PR #83520](https://github.com/kubernetes/kubernetes/pull/83520) 被 Revert 掉了，那么这个 PR 之前是为了修什么，又因为什么缺陷被 Revert 掉了？


##  修了，但没修好...

针对问题(1)，接着翻 git log 发现了另外一个 [PR #94316](https://github.com/kubernetes/kubernetes/pull/94316)。在这个 PR 中，它修订了 `isTooLargeResourceVersionError` 函数，修复了对 `TooLargeResourceVersionError` 这个异常的判断，并在注释中指出：在 1.17.0-1.18.5 的 K8S 版本中，无法正确的分辨当前的异常是否是 `TooLargeResourceVersionError`，即之前的 PR #92537 只有在 1.18.5 之后的 `kube-apiserver` 上是正常工作的。行吧，但是这也不能解释为何 0.22 的 `client-go` 不能和 1.16 的 `kube-apiserver` 工作啊：
1. PR #94316 在 0.18.9 进行了修复，0.22 肯定已经包含了这个修复了，即 1.17 以上的 `kube-apiserver` 不应该还有问题了；
2. 我们的 K8S 是 1.16，难道 1.16 和 1.17 在这个点上还有区别？

分析报错响应：
```JSON
--- # 1.20
{
   "kind":"Status",
   "apiVersion":"v1",
   "metadata":{
      
   },
   "status":"Failure",
   "message":"Timeout: Too large resource version: 900099999, current: 75601849",
   "reason":"Timeout",
   "details":{
      "causes":[
         {
            "reason":"ResourceVersionTooLarge",
            "message":"Too large resource version"
         }
      ],
      "retryAfterSeconds":1
   },
   "code":504
}

--- # 1.17
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "Timeout: Too large resource version: 90000000, current: 172897",
  "reason": "Timeout",
  "details": {
    "causes": [
      {
        "message": "Too large resource version"
      }
    ],
    "retryAfterSeconds": 1
  },
  "code": 504
}

--- # 1.16
{
   "kind":"Status",
   "apiVersion":"v1",
   "metadata":{
      
   },
   "status":"Failure",
   "message":"Timeout: Too large resource version: 9000, current: 2459",
   "reason":"Timeout",
   "details":{
      "retryAfterSeconds":1
   },
   "code":504
}
```

眼尖的同学已经发现了，1.16 没有 `details.cause` 字段， 1.17 则没有 `details.cause.reason` 字段。关于 1.17 缺失的这个字段，这也正是 PR #94316 进行修复以进行兼容的地方，而这个 fix 之后直到最新的 master (1.26) 都没有再改过这个地方了。也就是说，1.18 之后的 `client-go` 是无法正确处理 1.17 前 `kube-apiserver` 返回的 `TooLargeResourceVersionError` 异常的，这个地方存在一个兼容性问题。

那么关于修复方案，自然只有两条路：
1. 1.16 的 K8S 本来就不应该用 0.22 的 `client-go`，用回 1.16 兼容的 `0.16` 是合理的；
2. 想办法让 1.16 的 `kube-apiserver` 能够正确返回  `details.cause` 字段，从而兼容高版本的 `client-go`；

前者需要联系所有的组件，并进行 hotfix。对于一个已经发了版的版本，Hotfix 一个大横向复杂需求可能涉及到一堆沟通成本，毕竟相关产研的水平甚至不能正确理解包管理。而且面临 Java、Python 等众多的 SDK 语言版本，无法统一准确扫描影响访问和快速复现验收。所以压力很容易被转移到方案二上来，最简单的就是 cherrypick 相关的改动，如果改动比较简单又没有冲突，不失为一种快速的修复方案。

## 修不好了！

翻了下 git log，报错信息增加`details.cause` 的改动来自 [PR #72170](https://github.com/kubernetes/kubernetes/pull/72170), 而这个 PR 的标题则为 `Make resourceVersion parameter semantics consistent across all storage.Interface implementations`，看似和这个异常信息的修改毫无关系。Review 了下 PR 实现，它修改了 `etcd3` 存储层的实现，变更了 List 操作时候 RV 参数的语义，涉及到了一堆 Approver 的深度讨论，并且最后以 `I hope I don't regret this.` 的态度完成了代码的合并。 细究下去，这个 PR 背后是一个相当严重的 [缓存数据一致性Bug #59848](https://github.com/kubernetes/kubernetes/issues/59848) 与之相关的则是一篇复杂的 [设计文档](https://docs.google.com/document/d/1x3JXKwPTpum8S6bC-YXvoGS583iixuQSskEGukTAzcI) 。该 Bug 至今没有完全修复，而 PR #72170 则只是其中的一个 fix 而已。当仔细查看这个设计文档的时候，则会发现所有的问题都汇聚在了这里，之前造成兼容性问题的 PR #83520 也是故事的一部分。

这个Bug如果长话短说就是：客户端的缓存在HA部署的 `kube-apiserver` 时候，可能会发生时间倒退，继而导致最终行为上的错误。 这是一个数据一致性方面的根本性错误，打破了 API 基础原则中的线性一致性，会导致一连串的问题：比如可能会导致在不同节点出现两个同名的 Pod，这个的最坏情况（有状态业务）可能会使得业务数据丢失/损坏。这个 Bug 的细节非常复杂和有趣，详细的细节可以查看 [设计文档](https://docs.google.com/document/d/1x3JXKwPTpum8S6bC-YXvoGS583iixuQSskEGukTAzcI) 和 [原始 Bug Report #59848](https://github.com/kubernetes/kubernetes/issues/59848) ，我这里大概解释下我的理解。

由于 `kube-apiserver` 中缓存的存在，且客户端 List 请求的时候如果带的参数是 RV=0，则缓存中的数据会被直接返回（这个数据可能比实际 etcd 中的旧），这就导致如下场景：
1. 假设在 $t_1$ 时间在节点 $N_1$  上存在 Pod (uid=foo);
2. 这个 Pod 被删除（如滚动更新），$N_1$ 的 Kubelet 将 Pod (uid=foo) 停止，并删除其在 `kube-apiserver` 的元数据；
3. 控制器 (如 Statefulset Controller) 看到 Pod 被删除后不满足 replicas，故重新创建同名的 Pod (uid=bar);
4. 在 $t_2$ 时间在节点 $N_2$  上的 Kubelet 看到这个 Pod (uid=bar) 并将其启动;
5. 在 $t_1$ 时间在节点 $N_1$  上的 Kubelet 发生了异常退出并重启，重新初始化 Reflactor。这个过程中，会发起一次 List-Watch操作。但是由于存在多个 `kube-apiserver`，这次 List 返回的结果可能来自于其他的`kube-apiserver`实例，且该实例的缓存尚未追上最新的状态，故他将 Pod (uid=foo) 返回给了  $N_1$  上的 Kubelet，导致之前已经被删除的  Pod (uid=foo) 在 $N_1$ 被重新创建并拉起。

而这个问题经过讨论，总结来说它的触发则有可能有两种情况： relist 和 restart：
1. relist 代表客户端 Reflactor 并没有重初始化，还保留有之前的状态（如上次Watch到的最后的 RV 版本号）。这时候由于网络的抖动，或者请求的超时等等情况的时候，Reflacctor 会重新发起 List-Watch 操作，此时由于之前的 RV 还没丢失，因此还有可能可以利用之前的 RV 保持一致性。
2. restart 代表客户端 Reflactor 被完全推倒重新初始化，多见于类似进程重启的时候。此时之前内存里的的状态数据已经完全丢失，故没有办法提供上次 Watch 到的 RV 来保证一致性了。

还记得之前提到的 PR #83520 么？就是在 1.17 中被 Revert 掉，在 1.18 中引入 Bug 的那个PR，并在 1.18.9 才最终完全修复的那个 PR，并且导致至今为止都和 1.16 不兼容的那个 PR。这个 PR 就是着重解决了 relist 的这种情况，将之前 relist 的时候总是 `RV=0` 修改成了已知最新的 RV，这样就能保证重新 ListWatch 返回的数据总是比上次看到的新。但是已知的最新 RV 可能已经不能用了，比如发生了之前所说的，RV 已经从缓存中被淘汰了，这时候 API 会返回 `410 Gone` 。如果此时坚持继续使用这个 RV 进行重试，因为过去的版本不会再回来，因此客户端就会陷入死循环。因此这个地方需要对这个异常进行判断，在这种情况发生的时候，需要将 RV 参数置空（注意不是置0），强制绕过缓存直接从 etcd 获取数据。不巧的是，正如前面所说，已知的这个RV 除了太旧以外还可能太新，这个 PR 显然并未正确处理太新的情况 ，遗漏了对 `Too large resource version` 异常的处理，触发了客户端的死循环。

对于 restart 的时候，似乎没有很好的办法来解决。上次观察到的 RV 在重启后将丢失，持久化的话也找不到一个安全的持久化方式。如果在重启后第一次请求直接从 etcd 中获取，在大规模集群中将直接打爆 `kube-apiserver`。为此，社区寄希望于能够在 RV=0 的时候也能够从缓存中返回。对此 `kube-apiserver` 的维护者对 etcd 进行了加强，在 etcd 3.5 的 Watch API 上扩展了一个 `RequestProgress` 语义，使之能够很低的成本获取当前 etcd 中全局最新的 RV。当客户端发起 RV=0 的 List 请求的时候，会等到缓存追到这个最新的 RV 之后再实际返回，从而保证了不会返回旧的数据。

以上是背景知识，那么为什么 [PR #72170](https://github.com/kubernetes/kubernetes/pull/72170) 需要修改 List 操作 RV参数的语义呢？之前 etcd3 的 List 实现和 WatchCache 的实现不一致：在 WatchCache 的实现中，如果给定了一个非零的 RV，则返回的内容可能比给定的更大，而 etcd3 的实现则是必定返回的是当前给定的版本。上面对 relist / restart 的 fix 都依赖于给定的 RV使得返回更新的内容。但是如果 WatchCache 和 etcd3 对这块的实现不同，则意味着情况会变的复杂：对于客户端，服务端的缓存开启与否会导致 API 对外的行为不同。如果开启了缓存，当 RV 过小的时候能够正常返回一个当前缓存的版本，而 etcd3 实现如果这个 RV 被压缩掉了则会返回 410 Gone。一个行为不稳定的 API 可能会引发很多问题，所以 PR #72170 被提了出来作为之后 fix 的前置条件。

那么 PR #72170 做了什么呢？简单来说就是将 `storage.Interface` 这个接口的两个实现：WatchCache
和实际的 etcd3 实现做了行为上的统一。由于历史原因，这两者的行为并不完全相同。但是经过大佬们的讨论，这个修改在大多数情况下是安全的，因为：
1. 直接访问 etcd3 的时候，虽然忽略了给定的 RV。但是由于执行的 `quorum read` ，因此得到的数据是强一致保证的，因此必定比客户端已知的 RV 新；
2. 缓存开启的时候（默认开启），如果 RV 给定了，则请求一定从 cache 返回了，etcd3 的实现在链路上不会走到；

基于此，在评估了风险并浏览了所有树内代码后，PR #72170 的风险被认为可控，最后冒险进行了合并。这就是这个 PR，为 `TooLargeResourceVersionError` 定义了单独的错误类型，作为实现的一个副作用新增了 `details.cause.message` 字段。 那么为什么这个 PR 在 1.17 被回滚了呢？


### 被回滚的 PR #72170

在 1.17 发布后，有一个 [Issue #86483](https://github.com/kubernetes/kubernetes/issues/86483) 被汇报上来表明在升级 1.17.0 之后如果重启 `kube-apiserver` 会产生显著的惊群效应。由于修复的过程较为复杂和曲折讨论了很久，在有结论前由于风险较大直接在 [PR #86824](https://github.com/kubernetes/kubernetes/pull/86824) 在 1.17 版本内进行了回滚。具体惊群的路径如下：

1.  HA 集群中的 `kube-apiserver` 被重启后，由于之前 API 保持着的 Watch 长链接断开，使得 Client 发生 relist 操作；
2. 在多个 `kube-apiserver` 部署的时候，由于各实例的启动时间不同，其内部的 WatchCache 的 RV 是不同的，因为这个值来自于当时 etcd 内最大的 RV 值。假设 3 个 API 内的 RV 值分别是 v1、v2、v3，etcd 内全局最大的 RV 为 v100；
3. v1 的 `kube-apiserver` 重启后内部的 RV 更新到最新 v100，而之前和它连接的客户端带着之前已知的 v1 来访问 v2、v3 的 kube-apiserver 此时往往问题不大；
4. v3 的 `kube-apiserver` 重启后，而之前和它连接的客户端带着之前已知的 v3 去访问之前已经完成重启的 `kube-apiserver`，此时 `kube-apiserver` 最低认识的版本为 v100，则返回 401 Gone；
5. 由于 PR #72170 的改动，如果发生 Gone 异常的时候在重新发起的 List 请求中会将 RV 清空，因此这部分请求是无法被 WatchCache 缓存的，会被直接打进 etcd。注意这部分请求不止对最后重启的 `kube-apiserver` 是个问题，因为没有 RV 所以对于所有的 `kube-apiserver` 实例都是无法缓存的；
6. 高负载导致之前能正确处理的请求进入限流，导致重要请求比如 `lease` 失败，触发更多的组件触发 relist，进一步推高负载，最终导致雪崩；

这个问题于 [PR #86430](https://github.com/kubernetes/kubernetes/pull/86430/files) 中得到修复，这个 fix 修改了位于客户端的 Reflector，修复的手段不能说是 Dirty Hack 的话，说 Hack 一点也不过分。修改主要集中在以下两点：
1. 在配置了 RV != 0 的时候，将 ListOption 中的 PageSize 无效化；
2. 如果之前 List 返回的结果里出现过分页，则关闭 (1) 的这个行为；

```Go
list, paginatedResult, err = pager.List(context.Background(), options)
if isExpiredError(err) {
	r.setIsLastSyncResourceVersionExpired(true)
	// Retry immediately if the resource version used to list is expired.
	// The pager already falls back to full list if paginated list calls fail due to an "Expired" error on
	// continuation pages, but the pager might not be enabled, or the full list might fail because the
	// resource version it is listing at is expired, so we need to fallback to resourceVersion="" in all
	// to recover and ensure the reflector makes forward progress.
	list, paginatedResult, err = pager.List(context.Background(), metav1.ListOptions{ResourceVersion: r.relistResourceVersion()})
}
close(listCh)
```

要说清楚事情，无奈贴段代码在这。在修复之前，第一行的这个 List 会发出带  `RV=1&limit=500` 参数的请求。由于这个RV已经很久了，所以这个请求会返回 `410 Gone` ；于是将 RV 置空之后重新发起 List 请求，就是这个第二次请求打穿了缓存，打爆了 API。那么在修复之后呢？基于上面提到的，这个 PR 只不过关闭了分页功能，因此 `limit`  这个参数就不存在了，但是发出去的请求 RV 还是 1，这时候不也应该返回 `410 Gone`，然后继续一样的故事把 API 打爆么？事情就吊诡在这里，这个 PR 依赖了 WatchCache 的一个特定行为：**如果 List 请求提供了 RV 且开启了分页，则缓存无效**。这么设计的原因大概是因为由于缓存只有一个版本，所以无法在实现之后的 Continue 操作以获取第二页。所以啊，在分页关闭后第一个 List 请求就可以被 WatchCache 正确处理了，而对于 Cache 而言它只会保证返回的版本比提供的 RV 新，提供的 RV 再旧也不可能会返回 `410 Gone`，因此也不会去执行第二次昂贵的重试了。而至于上面提到的关闭 (1) 的这个行为的目的，则是为了将这个 Hack 尽可能缩小影响面，就不多赘述了。

这个惊群问题的 Fix 过程，从一开始尝试从服务端进行修复再到客户端这边关闭分页，最后添加额外的判断关闭 Hack 行为，整个过程经过了大量讨论。我看到的是：一方面 K8S 作为一个发布了 20 余个版本的大型项目，已经有了不少历史包袱，导致每个行为变更都可能影响到很多人，因此每个修复都显得战战兢兢的，很多人即使是 Approver 都不一定清楚整个逻辑路径，到最后甚至不惜用一些 Hack 来实现问题修复。而另外一方面，过程中的讨论都是较为清晰和专业的，即使是线下讨论也会将讨论的文档和结论同步到 Github 给后人追踪。


## 现在修的怎么样了？

至此，这个故事还没完全解决，还有两朵乌云：
1. `TooLargeResourceVersionError` 即使是在 1.18 的 API 和 SDK 组合下还是会继续发生，只不过能够重试后请求成功，而非 1.16 API + 1.18 SDK 的无限重试；
2. 在 Reflector 重启后，由于最新已经处理过 RV 的丢失，仍然可能会出现拿到『回到过去』的旧数据；

为了解决这个问题，K8S 的 API 的维护者在 etcd [PR #9869](https://github.com/etcd-io/etcd/pull/9869) 了 `RequestProgress` 的功能。通过这个 API，可以让 etcd 的消费者通过很廉价的方式获取到当前 etcd 中最大的 RV。有了这个 RV，之后访问 API 的 WatchCache 的时候，就能够在不提供 RV 的情况下使用这个从 etcd 侧拿到的 RV，等到缓存追到这个版本后，可靠地返回最新的数据。通过这个机制：
1. 这样多实例时候 `kube-apiserver` 就算某组 G/R 已经很久没有更新，Cache 的 RV 也可以得到有效的更新；
2. Reflector 重启后，即使 RV 丢失请求无法携带 RV，获得的数据也必定是发起请求时间之后的 etcd 内的数据，类似执行了非常廉价的 `qruorm read`；

值得一提的是，虽然这个 Bug 在 2018 年就已经被发现并有了上述的相关设计 [KEP #2340](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2340-Consistent-reads-from-cache)，但是这个功能截止目前还没有落地和实现，但其最新的设计可以在 [KEP #3157](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/3157-watch-list) 中找到，目前似乎是计划在 1.26 实现（1.26 已经发布，看来已经鸽了）。

除此以外，还有几个 KEP 是用来缓解/修复上述问题的：
1. [KEP #2523](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2523-consistent-resource-versions-semantics) : 这个提案看起来有点像是  [PR #72170](https://github.com/kubernetes/kubernetes/pull/72170) 的后续（猜测），相关 Code 在 1.19 中得到实现 ([PR #91505](https://github.com/kubernetes/kubernetes/pull/91505))。 由于没有FeatureGate控制，应该在 1.19 就可以用了；
2. [KEP #1904](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/1904-efficient-watch-resumption):  这个提案使用了 etcd 的 `WithProgressNotify` 参数，使得 etcd 周期性的广播当前最新的 RV。以此为基础，API 将这个 RV 通过 BookMark 的方式推送至客户端，从而更新 Reflector 内的 RV，进而大幅度降低在 API 重启的时候导致的 relist 问题。这个功能在 [PR #94364](https://github.com/kubernetes/kubernetes/pull/94364) 得到实现，以 FeatureGate `EfficientWatchResumption` 控制，在 1.21 进入 Beta；


# 决定 & 结局

至此，我觉得这个问题的现象、根因、背景扩展都已经了解的差不多了，应该足够做决定了。那么收一下，由于我们的环境是 1.16，让客户端去改可能不太可能了：
1. 全面排查客户端没有足够的资源，无法推动；
2. 1.16 的 `client-go` 的确存在 relist / restart 后会读到老数据的问题，回滚之后如果遇到问题还得背锅；

如果 `kube-apiserver` 进行修改的话：
1. 修改范围应该是在  [PR #72170](https://github.com/kubernetes/kubernetes/pull/72170) 之内，简单 cherrypick 了一下冲突可控，属于可以相对比较简单完成 cherrypick 的那种；
2. 这个 PR 做了两个事情：(1 修改了 etcd3 存储实现里对 RV 参数的语义，(2 给错误响应的  `details` 里加了 `cause` 字段；
3. 其中，语义的变化应该不会反应到我们的使用场景上：

> There is no in-tree code relying on it. We can't entirely eliminate the possibility there is out-of-tree code relying on it. The conditions they'd have to meet are quite specific: (1) they've disabled the watch cache, or are using events, and (2) they're performing a `List(... ResourceVersion: SomeSpecificRevision)` request and (3) they cannot tolerate receiving a more recent revision than the one they requested.

   因为我们都启用了 `WatchCache`，因此这部分 API 的返回之前都是从 Cache 返回的，而实际 etcd 的 List 语义应该没有启用过，故应该就算包含这部分改动也没用关系。

4. 由于 `cause` 字段在 API 定义中一直都有，如果只在这个异常发生的时候多返回一个字段：即手动cherrypick 这个改动里的一小部分应该可行的，只需要在 [reflector.go#L307](https://github.com/jpbetz/kubernetes/blob/release-1.16/staging/src/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go#L307) 修改一行：
```patch
- return errors.NewTimeoutError(fmt.Sprintf("Too large resource version: %v, current: %v", resourceVersion, w.resourceVersion), 1)
+ err := errors.NewTimeoutError(fmt.Sprintf("Too large resource version: %v, current: %v", resourceVersion, w.resourceVersion), 1)
+ err.ErrStatus.Details.Causes = []metav1.StatusCause{{Message: tooLargeResourceVersionCauseMsg}}
+ return err
```

可见，这个风险是最小的，且完整的 cherrypick 似乎并没有额外的受益，暂时先这样吧。
