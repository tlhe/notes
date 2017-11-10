#索引重建升级

Elasticsearch只能使用前一个主版本（major）创建的索引数据。例如，Elasticsearch 5.x能使用Elasticsearch 2.x创建的索引，但不能使用Elasticsearch 1.x或更早版本创建的索引。
>注意
Elasticsearch 5.x在存在老索引数据时会启动失败。

如果您正在运行一个包含了2.x版本创建的索引库的Elasticsearch 2.x集群，在升级到5.x之前你需要删除这些索引库或重新索引它们。参见 __Reindex in place__。

如果你运行的是Elasticsearch 1.x的集群，你有两个选择：
- 一种是升级到2.4.x，重建老的索引库，然后再升级到5.x。参见Reindex in place。
- 另一种是创建一个新的5.x集群，并使用reindex-from-remote来从1.x集群导入索引库。参见使用 __reindex-from-remote升级__。

>__基于时间的索引库与保留周期__
对于很多基于时间的索引库场景，不不太需要关心从1.x升级到5.x。通常基于时间的数据随着时间的推移数据都会变得没那么有趣，一旦超出保留周期老的索引库就可以删除掉。
用户可以在这个位置一直使用2.x直到所有1.x的索引库被全部删掉，然后再升级到5.x。

##Reindex in place

最简单的重建老索引(1.x)的方法是使用Elasticsearch迁移插件，你需要先升级Elasticsearch到2.3.x或2.4.x。
迁移插件提供的重建索引工具工作流程如下：
- 创建一个新的索引库，名称采用老的索引库名字拼接新的Elasticsearch版本号（例如：my_index-2.4.1），从新的索引库拷贝所有的映射与设置，新索引库禁用了冲刷、并且副本数被设置为0。
- 设置老索引库为只读，确保没有数据写入到老索引库。
- 重新索引老索引库的所有文档到新索引库。
- 根据老索引库的值重新设置新索引库refresh_interval与number_of_replicas的值，然后等待索引库状态变为green。
- 把老索引库的所有别名添加到新索引库。
- 删除老索引库。
- 用老索引库的名称作为别名添加到新索引库。例如：添加别名my_index指向索引库my_index-2.4.1。

过程结束后，你将会有一个新的2.x的索引库，并且它能被5.x的Elasticsearch集群使用。

## 使用reindex-from-remote升级

如果你有一个1.x的集群想升级到5.x，但是又不想先迁移到2.x，你可以使用reindex-from-remote。
>重要
Elasticsearch包含向后兼容的代码，允许索引库从上一个major版本升级到当前的major版本。在从1.x直接升级到5.x时，你必须自己解决所有向后兼容的问题。

你需要在已有的1.x集群旁边安装一个5.x的集群，5.x的集群要能够访问1.x集群的REST API。
对于每一个你想传输到5.x集群的1.x索引库，你需要：
- 在5.x创建一个新的索引，并使用适当的映射与设置。设置refresh_interval为-1，以及number_of_replicas为0来快速完成重建索引。
- 使用reindex-from-remote拉去1.x的索引库到5.x的新索引库。
- 如果你使用后台任务来重建索引(设置wait_for_completion为false)，重建索引的请求将会返回一个task_id，可以通过task API来监控重建索引的进度：GET _tasks/TASK_ID。
- 一旦重建索引完成，设置refresh_interval与number_of_replicas为希望的值(默认为30s和1).
- 一旦新索引库完成了副本，你可以删除老的索引库。

5.x的集群刚开始可以是一个小的集群，你可以根据你从1.x集群迁移到5.x集群的规模来逐步的增加节点。