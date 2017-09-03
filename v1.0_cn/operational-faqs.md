# 操作 FAQ

## 为什么当我尝试在后台启动进程时它挂起了？

第一个要问的问题是，你是否曾经使用同一个数据目录运行过一个多节点的集群。如果没有，那么你应该查看我们的[集群设置拍错文档](cluster-setup-troubleshooting.md)。如果你曾经启动并关闭了一个多节点集群，现在尝试将其重新在线，那么你问对了问题。

为了保存你的数据一致，CockroachDB 仅在至少它的大部分节点运行时才工作。这意味着，如果三节点集群中仅有一个节点运行，这一个节点不能做任何事。[`cockroach start`](start-a-node.md) 的 `--background` 标志使得启动命令等待直到节点完全初始化并能够开始服务查询。

这两个事实一起意味着，`--background` 标志会造成 `cockroach start` 挂起直到大部分节点运行。为了重启你的集群，你应该或者使用多个终端窗口，使得你能够同时启动多个节点，或者在后台使用你的 shell 功能启动每个节点（即 `cockroach start &`）而不是 `--background` 标志。

## 为什么没有流量而内存的使用量在增加？

如果你在计算机上启动了一个 CockroachDB 节点并让其运行了几个小时或几天，你可能会祝你到它的内存使用量在达到你的计算机总内存的 25% 的平稳值之前稳定增长了一段时间。这是预期的行为 -- 像大多数数据库一样，CockroachDB 将最近访问的数据缓存在内存中，使得它能够提供更快的读，而[它的定期写时序数据](#why-is-disk-usage-increasing-despite-lack-of-writes) 使得缓存的大小增长直到达到配置的限制。缓存大小的限制默认为计算机内存的 25%，但是可以通过在运行 [`cockroach start`](start-a-node.md) 时设置 `--cache` 标志来控制。

## 为什么没有写而磁盘的使用量在增加？

用于驱动管理 UI 中的图的时序数据存储在集群中并积累 30 天后开始被截断。其结果是，一个集群启动后的大约 30 天内，即使你自己没有写入数据，你将看到集群中磁盘使用量和域数量的稳定增长。

对于 1.0 发布版，没有办法改变时序数据开始被截断的天数。然而，一个解决办法是，你可以使用环境变量 `COCKROACH_METRICS_SAMPLE_INTERVAL` 启动每一个节点，设置高于高于默认值 `10s` 以存储较少的数据点。例如，你可以将其设置为 `1m`，只在每分钟收集数据，这将导致比默认设置存储少六倍的时序数据。

## 为什么 CockroachDB 默认收集匿名的集群使用细节？

收集关于 CockroachDB 在真实世界中的使用信息帮助我们优先排列产品特性的开发。我们选择"选择加入"作为默认以强化我们从收集中得到的信息，但是我们也仔细努力只发送匿名的、聚合的使用统计数据。哪些信息被发送以及如何选择退出的详细情况，见[诊断报告](diagnostics-reporting.md)。

## 当节点时钟没有正确同步时会发生什么？

CockroachDB 需要中等准确的时间以保持数据一致性，所以在每个节点上运行 [NTP](http://www.ntp.org/) 或者其他的时钟同步软件是重要的。

CockroachDB 的默认最大被允许的时钟偏移是 500ms。当一个节点检测到它的时钟偏移，相对于其他节点，是最大允许值的一半或更多时，它自发停机。这远未达到最大偏移（默认为 500ms），超出了最大偏移，[可序列化一致性](https://en.wikipedia.org/wiki/Serializability)不能保证，而且陈旧的读和写偏可能发生。 在每个节点上运行 NTP 或者其他的时钟同步软件，超出最大偏移和发生这类异常的风险非常小，即使在运行良好的硬件上，如果没有运行同步软件，缓慢的时钟漂移也很常见，CockroachDB 安全地处理了这种情况。

一个需要注意的罕见情况是，当一个节点的时钟偏移突然超出了最大值，而节点还没有检测到。尽管极端不可能发生，这是可能的，例如，在 VM 中运行 CockroachDB，VM 管理器觉得将 VM 迁移到运行在不同时间的不同的硬件上。在这种情况下，在节点的时钟变得不同步和节点自发停机之间，可能有一个小的时间窗口。在这个时间窗口期间，有可能客户端读了陈旧的数据并写入了从陈旧读中派生的数据。

## 另见

- [产品 FAQ](frequently-asked-questions.md)
- [SQL FAQ](sql-faqs.md)