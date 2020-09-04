# Pod的生命周期

Pod是k8s调度的最小工作单元。Pod 封装了应用程序容器（或者在某些情况下封装多个容器）、存储资源、唯一网络 IP 以及控制容器应该如何运行的选项。

理想情况下，不会直接在集群里创建pod而是使用更高层次的抽象资源，如Deployments, Replica Sets, Daemon Sets, Stateful Sets, or Jobs。通常情况下定位解决问题才会直接和pod交互，因此理解pod是很重要的。

Pod遵循一个预定义的生命周期，起始于Pending阶段，如果至少其中有一个主要容器正常启动，则进入Running，之后取决于Pod中是否有容器以失败状态结束而进入Succeeded或者Failed阶段。

在Pod运行期间，kubelet能够重启容器以处理一些失效场景。 在Pod内部，Kubernetes跟踪不同容器的状态并处理可能出现的状况。

在Kubernetes API中，Pod包含规约部分和实际状态部分。Pod对象的状态包含了一组Pod状况（Conditions）。 如果应用需要的话，你也可以向其中注入自定义的就绪性信息。

Pod 在其生命周期中只会被调度一次。一旦Pod被调度（分派）到某个节点，Pod会一直在该节点运行，直到 Pod停止或者被终止。

### Pod 生命期

和一个个独立的应用容器一样，Pod也被认为是相对临时性（而不是长期存在）的实体。Pod会被创建、赋予一个唯一的ID（UID）， 并被调度到节点，并在终止（根据重启策略）或删除之前一直运行在该节点。

如果一个节点死掉了，调度到该节点的Pod也被计划在给定超时期限结束后删除。

Pod自身不具有自愈能力。如果Pod被调度到某节点而该节点之后失效，或者调度操作本身失效，Pod会被删除；与此类似，Pod无法在节点资源耗尽或者节点维护期间继续存活。Kubernetes使用一种高级抽象，称作控制器，来管理这些相对而言可随时丢弃的Pod实例。

任何给定的Pod（由UID定义）从不会被“重新调度（rescheduled）”到不同的节点；相反，这一Pod可以被一个新的、几乎完全相同的Pod替换掉。如果需要，新Pod的名字可以不变，但是其UID会不同。

如果某物声称其生命期与某Pod相同，例如存储卷， 这就意味着该对象在此Pod（UID亦相同）存在期间也一直存在。 如果Pod因为任何原因被删除，甚至某完全相同的替代Pod被创建时，这个相关的对象（例如这里的卷）也会被删除并重建。

### Pod的生成

![birth of a pod](D:\01.new\07.doc\book\book\k8s\birth of a pod.jpg)

1. 利用kubectl或者别的API客户端向API server提交pod创建请求
2. API server把pod对象写入etcd中存储。一旦成功写入，就向API server和客户端发送确认消息。
3. API server现在开始代表etcd中状态的变化。
4. 所有的k8s组件利用watch不断的检查API Server来获取相关的变化。
5. kube-scheduler通过自己的watcher观察到API server上一个新创建的尚为绑定任何节点的Pod对象。
6. kube-scheduler给pod分配一个节点并更新API server。
7. 然后此更改将被更新到etcd存储。API server也将节点分配信息反应到Pod对象上。
8. 每个节点上的Kubelet同样运行着watcher。目标节点观察到一个新的pod分配给自己。
9. Kubelet调用docker在本节点上启动Pod，并把容器状态更新给API server。
10. API server把pod状态持久化到etcd。
11. 一旦成功写入etcd，API server发送确认消息到kubelet表示该事件被接收。

### Pod的终止

![termination of a pod](D:\01.new\07.doc\book\book\k8s\termination of a pod.jpg)

1. 用户执行命令删除pod。
2. 将API server中的pod对象更新上优雅删除时间，默认为30秒。
3. 一下几个动作是同时发生的：
   * 通过客户端命令查看时，pod的状态为Terminating 
   * 因为在步骤2中设置了优雅删除时间当观察到pod被标记为terminating时，kubelet就开始关闭pod的进程。
   * 端点控制器（Endpoint controller）监测到被删除的pod，并删除该pod服务的所有端点。
4. 如果pod定义了关闭前的回调钩子，它将被在pod内调用。如果在grace period超时后preStop钩子仍然在运行的话，步骤2将再次被调用设置一个2秒的grace period。
5. 向pod中的进程发送终止信号。
6. 当优雅周期超时，pod中运行的所有进程都会通过SIGKILL信号杀掉。
7. 通过设置grace period为0（立即删除），kubelet将完成在API server上删除pod。pod从API中消失并在客户端上不可见。

### Pod状态

#### Pod phase

Pod 的 status 字段是一个 PodStatus 对象，其中包含一个 phase 字段。

Pod 的阶段（Phase）是 Pod 在其生命周期中所处位置的简单宏观概述。 该阶段并不是对容器或 Pod 状态的综合汇总，也不是为了成为完整的状态机。

Pod 阶段的数量和含义是严格定义的。 除了本文档中列举的内容外，不应该再假定 Pod 有其他的 phase 值。

- 挂起（Pending）：Pod 已被 Kubernetes 系统接受，但有一个或者多个容器镜像尚未创建。等待时间包括调度 Pod 的时间和通过网络下载镜像的时间，这可能需要花点时间。
- 运行中（Running）：该 Pod 已经绑定到了一个节点上，Pod 中所有的容器都已被创建。至少有一个容器仍然在运行，或者正处于启动或重启状态。
- 成功（Succeeded）：Pod 中的所有容器都被成功终止，并且不会再重启。
- 失败（Failed）：Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止。也就是说，容器以非0状态退出或者被系统终止。
- 未知（Unknown）：因为某些原因无法取得 Pod 的状态，通常是因为与 Pod 所在主机通信失败。

如果某节点死掉或者与集群中其他节点失联，Kubernetes 会实施一种策略，将失去的节点上运行的所有 Pod 的 phase 设置为 Failed。

#### 容器状态
Kubernetes 会跟踪 Pod 中每个容器的状态，就像它跟踪 Pod 总体上的阶段一样。 你可以使用容器生命周期回调 来在容器生命周期中的特定时间点触发事件。

一旦调度器将 Pod 分派给某个节点，kubelet 就通过容器运行时开始为 Pod 创建容器。 容器的状态有三种：Waiting（等待）、Running（运行中）和 Terminated（已终止）。

要检查 Pod 中容器的状态，你可以使用 kubectl describe pod <pod 名称>。 其输出中包含 Pod 中每个容器的状态。

每种状态都有特定的含义：

#####  Waiting （等待）
如果容器并不处在 Running 或 Terminated 状态之一，它就处在 Waiting 状态。 处于 Waiting 状态的容器仍在运行它完成启动所需要的操作：例如，从某个容器镜像 仓库拉取容器镜像，或者向容器应用 Secret 数据等等。 当你使用 kubectl 来查询包含 Waiting 状态的容器的 Pod 时，你也会看到一个 Reason 字段，其中给出了容器处于等待状态的原因。

##### Running（运行中）
Running 状态表明容器正在执行状态并且没有问题发生。 如果配置了 postStart 回调，那么该回调已经执行完成。 如果你使用 kubectl 来查询包含 Running 状态的容器的 Pod 时，你也会看到 关于容器进入 Running 状态的信息。

##### Terminated（已终止）
处于 Terminated 状态的容器已经开始执行并且或者正常结束或者因为某些原因失败。 如果你使用 kubectl 来查询包含 Terminated 状态的容器的 Pod 时，你会看到容器进入此状态的原因、退出代码以及容器执行期间的起止时间。

如果容器配置了 preStop 回调，则该回调会在容器进入 Terminated 状态之前执行。

#### 容器重启策略

Pod 的 spec 中包含一个 restartPolicy 字段，其可能取值包括 Always、OnFailure 和 Never。默认值是 Always。

restartPolicy 适用于 Pod 中的所有容器。restartPolicy 仅针对同一节点上 kubelet 的容器重启动作。当 Pod 中的容器退出时，kubelet 会按指数回退 方式计算重启的延迟（10s、20s、40s、...），其最长延迟为 5 分钟。 一旦某容器执行了 10 分钟并且没有出现问题，kubelet 对该容器的重启回退计时器执行 重置操作。

#### Pod conditions

PodStatus中有一个表示pod是否通过检测的数组 PodConditions。PodCondition元素有6个可能的字段。

* lastProbeTime：表示该condition最近一次探测的时间戳。
* lastTransitionTime：最近一次状态变化的时间戳。
* message：指示状态变化可读的详细信息。
* reason：唯一的、一个单词的、CamlCase的表示该condition最近转变的原因。
* status：string字段，包括"True", "False", "Unknown"。
* type：包括下面几个可能的值
  * PodScheduled：Pod是否已经调度到特定的节点。
  * Ready：Pod可以对外提供服务，并且应该被添加到所有匹配的负载池中。
  * Initialized：所有 init containers都已经成功启动
  * ContainersReady：pod中所有容器都准备就绪。

### Pod 就绪态

你的应用可以向 PodStatus 中注入额外的反馈或者信号：Pod Readiness（Pod 就绪态）。 要使用这一特性，可以设置 Pod 规约中的 readinessGates 列表，为 kubelet 提供一组额外的状况供其评估 Pod 就绪态时使用。

就绪态门控基于 Pod 的 status.conditions 字段的当前值来做决定。 如果 Kubernetes 无法在 status.conditions 字段中找到某状况，则该状况的 状态值默认为 "False"。

```
kind: Pod
...
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready                              # 内置的 Pod 状况
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"        # 额外的 Pod 状况
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
...
```

你所添加的 Pod 状况名称必须满足 Kubernetes [标签键名格式](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set)。

### Pod 就绪态的状态

命令 kubectl patch 不支持修改对象的状态。 如果需要设置 Pod 的 status.conditions，应用或者 [Operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) 需要使用 PATCH 操作。 你可以使用 Kubernetes 客户端库 之一来编写代码，针对 Pod 就绪态设置定制的 Pod 状况。

对于使用定制状况的 Pod 而言，只有当下面的陈述都适用时，该 Pod 才会被评估为就绪：

* Pod 中所有容器都已就绪；

* readinessGates 中的所有状况都为 True 值。

当 Pod 的容器都已就绪，但至少一个定制状况没有取值或者取值为 False， kubelet 将 Pod 的状况设置为 ContainersReady。

### 容器探针

探针 是由 kubelet 对容器执行的定期诊断。 要执行诊断，kubelet 调用由容器实现的 Handler （处理程序）。有三种类型的处理程序：

* ExecAction： 在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。

* TCPSocketAction： 对容器的 IP 地址上的指定端口执行 TCP 检查。如果端口打开，则诊断被认为是成功的。

* HTTPGetAction： 对容器的 IP 地址上指定端口和路径执行 HTTP Get 请求。如果响应的状态码大于等于 200 且小于 400，则诊断被认为是成功的。

每次探测都将获得以下三种结果之一：

* Success（成功）：容器通过了诊断。
* Failure（失败）：容器未通过诊断。
* Unknown（未知）：诊断失败，因此不会采取任何行动。

针对运行中的容器，kubelet 可以选择是否执行以下三种探针，以及如何针对探测结果作出反应：

* livenessProbe：指示容器是否正在运行。如果存活态探测失败，则 kubelet 会杀死容器， 并且容器将根据其重启策略决定未来。如果容器不提供存活探针， 则默认状态为 Success。

* readinessProbe：指示容器是否准备好为请求提供服务。如果就绪态探测失败， 端点控制器将从与 Pod 匹配的所有服务的端点列表中删除该 Pod 的 IP 地址。 初始延迟之前的就绪态的状态值默认为 Failure。 如果容器不提供就绪态探针，则默认状态为 Success。

* startupProbe: 指示容器中的应用是否已经启动。如果提供了启动探针，则所有其他探针都会被 禁用，直到此探针成功为止。如果启动探测失败，kubelet 将杀死容器，而容器依其 重启策略进行重启。 如果容器没有提供启动探测，则默认状态为 Success。(1.14版本还没有)

如欲了解如何设置存活态、就绪态和启动探针的进一步细节，可以参阅 配置存活态、就绪态和启动探针。



#### 何时该使用存活态探针?
如果容器中的进程能够在遇到问题或不健康的情况下自行崩溃，则不一定需要存活态探针; kubelet 将根据 Pod 的restartPolicy 自动执行修复操作。

如果你希望容器在探测失败时被杀死并重新启动，那么请指定一个存活态探针， 并指定restartPolicy 为 "Always" 或 "OnFailure"。

#### 何时该使用就绪态探针?
如果要仅在探测成功时才开始向 Pod 发送请求流量，请指定就绪态探针。 在这种情况下，就绪态探针可能与存活态探针相同，但是规约中的就绪态探针的存在意味着 Pod 将在启动阶段不接收任何数据，并且只有在探针探测成功后才开始接收数据。

如果你的容器需要加载大规模的数据、配置文件或者在启动期间执行迁移操作，可以添加一个 就绪态探针。

如果你希望容器能够自行进入维护状态，也可以指定一个就绪态探针，检查某个特定于 就绪态的因此不同于存活态探测的端点。

> 说明： 请注意，如果你只是想在 Pod 被删除时能够排空请求，则不一定需要使用就绪态探针； 在删除 Pod 时，Pod 会自动将自身置于未就绪状态，无论就绪态探针是否存在。 等待 Pod 中的容器停止期间，Pod 会一直处于未就绪状态。
#### 何时该使用启动探针？
> FEATURE STATE: Kubernetes v1.16 [alpha]

对于所包含的容器需要较长时间才能启动就绪的 Pod 而言，启动探针是有用的。 你不再需要配置一个较长的存活态探测时间间隔，只需要设置另一个独立的配置选定， 对启动期间的容器执行探测，从而允许使用远远超出存活态时间间隔所允许的时长。

如果你的容器启动时间通常超出 initialDelaySeconds + failureThreshold × periodSeconds 总值，你应该设置一个启动探测，对存活态探针所使用的同一端点执行检查。 periodSeconds 的默认值是 30 秒。你应该将其 failureThreshold 设置得足够高， 以便容器有充足的时间完成启动，并且避免更改存活态探针所使用的默认值。 这一设置有助于减少死锁状况的发生。

### Pod 的终止

由于 Pod 所代表的是在集群中节点上运行的进程，当不再需要这些进程时允许其体面地 终止是很重要的。一般不应武断地使用 KILL 信号终止它们，导致这些进程没有机会 完成清理操作。

设计的目标是令你能够请求删除进程，并且知道进程何时被终止，同时也能够确保删除 操作终将完成。当你请求删除某个 Pod 时，集群会记录并跟踪 Pod 的体面终止周期， 而不是直接强制地杀死 Pod。在存在强制关闭设施的前提下， kubelet 会尝试体面地终止 Pod。

通常情况下，容器运行时会发送一个 TERM 信号到每个容器中的主进程。 一旦超出了体面终止限期，容器运行时会向所有剩余进程发送 KILL 信号，之后 Pod 就会被从 API 服务器 上移除。如果 kubelet 或者容器运行时的管理服务在等待进程终止期间被重启， 集群会从头开始重试，赋予 Pod 完整的体面终止限期。

下面是一个例子：

1. 你使用 kubectl 工具手动删除某个特定的 Pod，而该 Pod 的体面终止限期是默认值（30 秒）。

2. API 服务器中的 Pod 对象被更新，记录涵盖体面终止限期在内 Pod 的最终死期，超出所计算时间点则认为 Pod 已死（dead）。 如果你使用 kubectl describe 来查验你正在删除的 Pod，该 Pod 会显示为 "Terminating" （正在终止）。 在 Pod 运行所在的节点上：kubelet 一旦看到 Pod 被标记为正在终止（已经设置了体面终止限期），kubelet 即开始本地的 Pod 关闭过程。
   1. 如果 Pod 中的容器之一定义了 preStop 回调， kubelet 开始在容器内运行该回调逻辑。如果超出体面终止限期时，preStop 回调逻辑 仍在运行，kubelet 会请求给予该 Pod 的宽限期一次性增加 2 秒钟。

      > 说明： 如果 preStop 回调所需要的时间长于默认的体面终止限期，你必须修改 terminationGracePeriodSeconds 属性值来使其正常工作。

   2. kubelet 接下来触发容器运行时发送 TERM 信号给每个容器中的进程 1。

      > 说明： Pod 中的容器会在不同时刻收到 TERM 信号，接收顺序也是不确定的。 如果关闭的顺序很重要，可以考虑使用 preStop 回调逻辑来协调。

3. 与此同时，kubelet 启动体面关闭逻辑，控制面会将 Pod 从对应的端点列表（以及端点切片列表， 如果启用了的话）中移除，过滤条件是 Pod 被对应的 服务以某 选择算符选定。 ReplicaSets和其他工作负载资源 不再将关闭进程中的 Pod 视为合法的、能够提供服务的副本。关闭动作很慢的 Pod 也无法继续处理请求数据，因为负载均衡器（例如服务代理）已经在终止宽限期开始的时候 将其从端点列表中移除。
4. 超出终止宽限期线时，kubelet 会触发强制关闭过程。容器运行时会向 Pod 中所有容器内 仍在运行的进程发送 SIGKILL 信号。 kubelet 也会清理隐藏的 pause 容器，如果容器运行时使用了这种容器的话。
5. kubelet 触发强制从 API 服务器上删除 Pod 对象的逻辑，并将体面终止限期设置为 0 （这意味着马上删除）。
6. API 服务器删除 Pod 的 API 对象，从任何客户端都无法再看到该对象。

### 强制终止 Pod

> 注意： 对于某些工作负载及其 Pod 而言，强制删除很可能会带来某种破坏。

默认情况下，所有的删除操作都会附有 30 秒钟的宽限期限。 kubectl delete 命令支持 --grace-period=<seconds> 选项，允许你重载默认值， 设定自己希望的期限值。

将宽限期限强制设置为 0 意味着立即从 API 服务器删除 Pod。 如果 Pod 仍然运行于某节点上，强制删除操作会触发 kubelet 立即执行清理操作。

> 说明： 你必须在设置 --grace-period=0 的同时额外设置 --force 参数才能发起强制删除请求。

执行强制删除操作时，API 服务器不再等待来自 kubelet 的、关于 Pod 已经在原来运行的节点上终止执行的确认消息。 API 服务器直接删除 Pod 对象，这样新的与之同名的 Pod 即可以被创建。 在节点侧，被设置为立即终止的 Pod 仍然会在被强行杀死之前获得一点点的宽限时间。

如果你需要强制删除 StatefulSet 的 Pod，请参阅 从 StatefulSet 中删除 Pod 的任务文档。

#### 失效 Pod 的垃圾收集

对于已失败的 Pod 而言，对应的 API 对象仍然会保留在集群的 API 服务器上，直到 用户或者控制器进程显式地 将其删除。

控制面组件会在 Pod 个数超出所配置的阈值 （根据 kube-controller-manager 的 terminated-pod-gc-threshold 设置）时 删除已终止的 Pod（阶段值为 Succeeded 或 Failed）。 这一行为会避免随着时间演进不断创建和终止 Pod 而引起的资源泄露问题。