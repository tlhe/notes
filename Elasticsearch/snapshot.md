#快照和还原

快照和还原模块支持创建单独索引或整个集群的快照到远程仓库，像共享文件系统、S3或HDFS等。你可以非常方便地进行快照备份以及比较迅速地进行快照还原，但快照的还原需要Elasticsearch的版本能正确的读取索引文件。这意味着：
- 在2.x版本中创建的索引的快照可以恢复到5.x
- 在1.x版本中创建的索引的快照可以恢复到2.x
- 在1.x版本中创建的索引的快照不可以恢复到5.x

要恢复在1.x版本中创建的索引快照到5.x，你可以先将其恢复到2.x版本的集群，然后重建索引到5.x集群。因为是从原始数据存档的副本恢复, 这会比较耗时。

##仓库

在执行任何快照或还原操作之前，应在先在Elasticsearch创建一个快照仓库。指定仓库的类型等，详情请参阅下文。
```
PUT /_snapshot/my_backup
{
  "type": "fs",
  "settings": {
        ... repository specific settings ...
  }
}
```
一旦存储仓库被创建好，可使用下面的命令来查询其信息：
```
GET /_snapshot/my_backup
```
将返回：
```
{
  "my_backup": {
    "type": "fs",
    "settings": {
      "compress": true,
      "location": "/mount/backups/my_backup"
    }
  }
}
```
可以用逗号分隔库名来一次获取多个快照仓库的信息，支持通配符*。例如，获取以repo开头以及包含backup的所有仓库信息 可以使用下面的命令来获得：
```
GET /_snapshot/repo*,*backup*
```
如果没有指定仓库名称，或使用_all用作仓库名来查询，Elasticsearch将返回当前集群中注册的所有仓库信息：
```
GET /_snapshot
```
或者
```
GET /_snapshot/_all
```

##共享文件系统仓库

共享文件系统仓库（"type": "fs"）使用共享文件系统来存储快照。如果要创建一个共享文件系统类型的快照仓库，你需要在Elasticsearch集群的所有主节点与数据节点上挂载同一个共享文件系统到同一路径下。这个路径（或它的父目录中的一个）必须在集群所有主节点和数据节点上的path.repo设置项中进行配置。

如果共享文件系统被挂载到/mount/backups/my_backup，下面的配置应被添加到elasticsearch.yml文件中：
```
path.repo: ["/mount/backups", "/mount/longterm_backups"]
```
path.repo的配置支持微软的Windows UNC路径格式，但需要将服务器名与指定的共享路径前缀采用斜杠正确转义：
```
path.repo: ["\\\\MY_SERVER\\Snapshots"]
```
所有节点重新启动后，下面的命令可以被用于创建一个名为my_backup的共享文件系统仓库：
```
$ curl -XPUT 'http://localhost:9200/_snapshot/my_backup' -H 'Content-Type: application/json' -d '{
    "type": "fs",
    "settings": {
        "location": "/mount/backups/my_backup",
        "compress": true
    }
}'
```
如果存储仓库的location被设置为一个相对路径，那么仓库存储路径将基于path.repo设置的路径开始解析：
```
$ curl -XPUT 'http://localhost:9200/_snapshot/my_backup' -H 'Content-Type: application/json' -d '{
    "type": "fs",
    "settings": {
        "location": "my_backup",
        "compress": true
    }
}'
```
下面的一些配置将被支持：
|参数名|	描述|
| ------------- |-------------|
|location|快照存放的位置。必须设置，不能为空。|
|compress|是否开启快照文件的压缩。压缩仅处理元数据文件（索引的映射与配置信息）。数据文件不会被压缩。默认为true。|
|chunk_size|可以根据需要在创建快照过程中将大文件分解成块。块大小以字节为单位，或直接指定大小单位，例如：1g，10m，5k。默认为null（不限制块大小）。|
|max_restore_bytes_per_sec|每个节点在快照还原时的速率。默认为40mb每秒。|
|max_snapshot_bytes_per_sec|每个节点在快照创建时的速率。默认为40mb每秒。|
|readonly|标记仓库为只读。默认为false。|

##只读URL仓库
URL仓库（"type": "url"）可以被用作另一种只读方式去访问共享文件系统仓库的数据。需要指定url参数为共享文件系统仓库的根路径。下面的一些设置将被支持：
|参数名|	描述|
| ------------- |-------------|
|url|快照的位置。必须设置，不能为空。|

URL仓库支持以下协议： “HTTP”， “HTTPS”， “FTP”， “file”和“jar”。URL仓库在使用http:， https:和ftp:协议的路径时，必须指定允许的URL被列入白名单repositories.url.allowed_urls，此设置在主机、路径、查询参数和片段的地方支持通配符。例如：
```
repositories.url.allowed_urls: ["http://www.example.org/root/*", "https://*.mydomain.com/*?*#*"]
```
URL仓库在使用file:协议的路径时，路径只能指向配置在path.repo中的设置，配置方式类似于共享文件系统仓库。
##仓库插件
官方可用的其他仓库插件：
- [repository-s3](https://www.elastic.co/guide/en/elasticsearch/plugins/5.3/repository-s3.html)支持S3仓库
- [repository-hdfs](https://www.elastic.co/guide/en/elasticsearch/plugins/5.3/repository-hdfs.html)支持Hadoop环境的的HDFS仓库
- [repository-azure](https://www.elastic.co/guide/en/elasticsearch/plugins/5.3/repository-azure.html)支持Azure的仓库
- [repository-gcs](https://www.elastic.co/guide/en/elasticsearch/plugins/5.3/repository-gcs.html)支持谷歌云存储仓库
##仓库验证
当仓库创建完成后，会立即核实所有主节点与数据节点，以确保集群上所有节点上的功能。该verify参数可用于在创建或更新仓库时显式地禁用仓库验证：
```
PUT /_snapshot/s3_repository?verify=false
{
  "type": "s3",
  "settings": {
    "bucket": "my_s3_bucket",
    "region": "eu-west-1"
  }
}
```
验证过程也可通过运行下面的命令手动执行：
```
POST /_snapshot/s3_repository/_verify
```
它将返回验证成功的所有节点，或者是验证失败的消息。
##快照
一个仓库可以包含相同的集群的多个快照。快照需要在集群内标记成一个唯一的名称。以下是在my_backup的仓库中创建一个名为snapshot_1的快照指令：
```
PUT /_snapshot/my_backup/snapshot_1?
wait_for_completion=true
```
*wait_for_completion* 参数标识此请求是否需要等待快照创建完成，默认立即返回，不等待快照创建结束。在快照创建开始时，所有之前的快照信息都将被加载到内存，这意味着一个大仓库执行此命令可能需要几秒钟（或甚至几分钟）才能返回，即使该命令的*wait_for_completion*参数设置为false。

创建快照时，默认会将集群中所有打开和已开始的索引全部备份，也可以通过设置创建快照的参数体来为指定为哪些索引创建快照。
```
PUT /_snapshot/my_backup/snapshot_1
{
  "indices": "index_1,index_2",
  "ignore_unavailable": true,
  "include_global_state": false
}
```
快照包含的索引列表可以通过*indices*参数设置，支持多索引语法。快照请求还支持*ignore_unavailable*选项，设置它true会在快照创建过程中忽略不存在的索引错误，默认情况下*ignore_unavailable*参数未设置，在创建快照时如果缺少一个索引将会失败。通过设置*include_global_state*为*false*，可以在创建的快照中不存储集群的全局状态信息。默认情况下，如果参与快照的一个或多个索引没有可用的主分片，则整个快照将失败，这种行为可以通过设置*partial*为*true*来改变。

索引创建快照时是增量的处理时。在创建索引快照时，Elasticsearch会分析创库中已存储的快照信息，只复制上一次快照后发生的新增、或发生变化的文件。它允许多个快照保存在一个仓库中。创建快照过程采用非阻塞方式执行。正在执行创建快照的索引可以继续执行索引和搜索操作。但快照是表示在创建快照那一瞬间点的索引视图，所以在快照开始执行之后添加到索引将不存在于快照中。快照在主分片上会立即开始执行，快照开始后这个集群分片将不会迁移。1.2.0版本之前，在创建快照时如果群集有参与快照的索引的任何主分片搬迁或初始化动作，创建快照都将会失败。从1.2.0版本开始，Elasticsearch将等待分片搬迁或初始化完成之后再执行快照。

除了创建每个索引的备份外，快照也可以存储全局集群的元数据，其中包括群集的持久化设置和模板信息，临时设置和已创建的快照仓库将不会作为快照的一部分存储。

在一个集群中，同一时间只能有一个创建快照的进程被执行。虽然已经开始创建快照的分片不能移动到另一个节点，但可能会干扰集群的再平衡处理和分配过滤。Elasticsearch将只能在快照结束时，根据当前分配过滤设置和再平衡算法将一个分片移动到另一个节点。

一旦开始创建快照，可以通过如下命令来查询此快照的有关信息：
```
GET /_snapshot/my_backup/snapshot_1
```
此命令返回的快照相关信息包括起始和结束时间、创建快照的elasticsearch版本、包含的索引列表、快照的当前状态、以及在快照创建过程中发生故障的索引列表。快照状态如下：
|状态码|描述|
|------|-----|
|IN_PROGRESS|当前快照正在运行。|
|SUCCESS|创建快照完成并且所有分片都存储成功。|
|FAILED|创建快照失败，没有存储任何数据。|
|PARTIAL|集状态全局状态已储存，但至少有一个分片的数据没有存储成功。在返回的failure字段中包含了相关未正确处理的分片详细信息。|
|INCOMPATIBLE|快照是由一个老版本的elasticsearch创建，因此与集群的当前版本不兼容。|

跟仓库类似，多个快照可以在一个请求查询，还支持通配符：
```
GET /_snapshot/my_backup/snapshot_*,some_other_snapshot
```
下面指令介绍了如何查询仓库中的所有快照：
```
GET /_snapshot/my_backup/_all
```
如果一些快照不可用导致查询失败，可以通过设置布尔参数ignore_unavailable来返回当前可用的所有快照。

正在运行中的快照可以使用下面的命令来查询：
```
$ curl -XGET "localhost:9200/_snapshot/my_backup/_current"
```
快照可以使用下面的命令来从仓库中删除：
```
DELETE /_snapshot/my_backup/snapshot_1
```
当一个快照从版本库中删除，Elasticsearch将删除此快照关联的但不被其它快照引用的所有文件。一旦删除快照动作执行，此快照当前正在创建的进程将被终止并且已创建的文件都将被清理。因此，删除快照操作可以用于取消错误启动的长时间运行的快照操作。

可以使用下面的命令删除整个仓库：
```
DELETE /_snapshot/my_backup
```
当仓库被删除时，Elasticsearch只是删除快照的仓库位置引用信息。快照本身没有删除，并在原来的位置。
##还原
快照可以使用下面的命令来还原：
```
POST /_snapshot/my_backup/snapshot_1/_restore
```
默认情况下，快照中的所有索引都会被还原，集群的全局设置不会被还原。这可能是最好的还原选择，允许通过设置*indices*与*include_global_state*参数来控制还原集群的全局设置与索引。索引列表支持多索引语法。*rename_pattern*与*rename_replacement*选项可以在还原时通过正则表达式来重命名索引。设置*include_aliases*为*false*可以防止别名和关联的索引一起被还原。
```
POST /_snapshot/my_backup/snapshot_1/_restore
{
  "indices": "index_1,index_2",
  "ignore_unavailable": true,
  "include_global_state": true,
  "rename_pattern": "index_(.+)",
  "rename_replacement": "restored_index_$1"
}
```
还原操作可以在正常工作的集群上执行。但是，已存在的索引需要先关闭后才能还原，并要跟快照中的索引具有相同数目的分片。还原操作将自动打开已存在的关闭的索引，如果被还原的索引在集群中不存在，将创建新的索引。如果需要还原群集状态include_global_state（默认false），所有不存在的模板都会新增、已存在的会使用快照中的同名模板替换，持久化设置也将被添加到现有的持久化设置中。
##部分还原
默认情况下，如果参与操作的一个或多个索引没有可用的快照分片，整个还原操作都将失败。这可能是创建快照时一些分片备份失败，你依然可以通过设置partial参数为true来尽可能的去还原。请注意，只有备份成功的分片才会在这种情况还原，所有丢失的分片将被创建成一个空的分片。
##还原过程中更改索引设置
大多数的索引设置可以在还原过程中被重写。例如，下面的指令将在还原索引index_1时，不创建任何副本以及采用默认的索引刷新间隔：
```
POST /_snapshot/my_backup/snapshot_1/_restore
{
  "indices": "index_1",
  "index_settings": {
    "index.number_of_replicas": 0
  },
  "ignore_index_settings": [
    "index.refresh_interval"
  ]
}
```
请注意，有一些设置不能在还原时修改，如
*index.number_of_shards*。
##还原到不同集群
存储在快照中的信息是不依赖于特定的群集或群集名称的，因此有可能从一个集群还原另一个群集创建的快照。在开始还原前，需要在新的集群中创建一个包含了原快照文件的仓库，新的群集不必具有相同的大小或拓扑。然而，新集群的版本应该比用于创建快照的群集的版本更高或者相同（但只能跨1个重大更新的版本）。例如，你可以还原1.x版本的快照到2.x版本的集群，而不是1.x版本的快照到5.x版本的集群。

如果新的集群节点比原集群少，需要做一些额外的考虑。首先，必须确保新的集群有足够的容量来存储所有索引的快照。你可以在还原时配置减少副本的数量，这能帮助我们将快照还原到更小集群，也可以指定*indices*参数为快照中的索引的子集。在elasticsearch 1.5.0版本之前，还原持久设置是没有检查*discovery.zen.minimum_master_nodes*的设置，可能导致还原到一个小的集群时不兼容，因此可以在较小的集群中禁用此设置直到达到加入所需的主节点资格数。从1.5.0版本开始不兼容的设置将会被忽略。

如果原来的集群中的索引分配了特定节点的分片分配过滤，相同的规则将在新的集群中执行。因此，如果新的群集不包含与被还原的索引可以匹配的属性节点，这样的索引将不会被成功还原，除非在还原时修改这些索引的分配配置。
##快照状态
可以使用下面的命令来查询当前正在运行的快照列表的详细状态信息：
```
GET /_snapshot/_status
```
在这种格式，此命令将返回当前运行的所有快照的信息。通过指定仓库的名字可以限制返回特定仓库的快照信息：
```
GET /_snapshot/my_backup/_status
```
如果同时指定仓库名称和快照ID，此命令将返回指定快照的状态信息，即使它当前不是正在运行：
```
GET /_snapshot/my_backup/snapshot_1/_status
```
多个ID也被支持：
```
GET /_snapshot/my_backup/snapshot_1,snapshot_2/_status
```
##监控快照/还原进度
有几种方法可以用来监测快照创建与还原正在运行时的执行进度。在创建快照与还原时指定*wait_for_completion*参数来阻塞客户端，直到操作完成，这是一个可以用来收到操作完成通知的最简单方法。

快照操作还可以通过定期获取快照信息来监测：
```
GET /_snapshot/my_backup/snapshot_1
```
请注意，获取快照信息操作与创建快照操作使用相同的资源和线程池，因此在执行获取快照信息时如果创建快照的线程正在备份大的分片，可能导致在返回结果前等待可用的资源。在非常大的分片备份情况中等待的时间更明显。

为了获得更多的直接和完整的信息，可以用快照的status命令来查看：
```
GET /_snapshot/my_backup/snapshot_1/_status
```
快照信息指令仅返回正在进行的快照的基本信息，快照状态指令将返回当前参与备份的每个分片的完成状态。

还原过程是基于Elasticsearch标准恢复机制，因此标准的恢复监控服务可以用来监视还原的状态。当执行集群还原操作时通常会进入*red*状态，这是因为还原操作是从“恢复”被还原的索引的主片开始的。在此操作期间主片变得不可用，这表现在集群状态为*red*。一旦Elasticsearch主片被切换到恢复完成时，这时整个集群的状态将被切换成*yellow*并且开始创建所需数量的副本。一旦创建了所有必需的副本，集群切换到*green*状态。

集群的健康情况只是在还原过程中提供了一个比较粗的状态，你还可以通过使用[indices recovery](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-recovery.html)与[cat recovery](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-recovery.html)的API来获得更详细的恢复过程信息与索引的当前状态信息。
##停止当前正在运行的快照和还原操作
快照和还原框架只允许在同一个时间点只运行一个快照或一个还原操作。如果当前正在运行的快照被错误执行，或执行时间特别长，可以使用删除快照操作来终止它。删除快照操作将检查当前快照是否正在运行，如果确实如此，删除操作将停止该快照并从仓库中删除之前的快照数据。
```
DELETE /_snapshot/my_backup/snapshot_1
```
还原操作使用标准分片恢复机制。因此，任何当前正在运行的还原操作都可以通过删除正在还原的索引来取消。请注意，所有被删除的索引数据都将从集群中通过这一操作全部删除。
##快照和还原操作对集群阻塞的影响
许多快照与还原操作都会给集群和索引的阻塞带来影响。例如，创建和删除仓库需要写全局元数据。创建快照操作要求所有索引及其元数据、以及全局元数据是可读的。还原操作需要全局元数据可写。但索引级别的阻塞在还原过程中可以忽略，因为索引的还原本质上是新建索引。请注意，仓库内容不是群集的一部分，因此集群阻塞不会影响仓库内部的操作，譬如在一个已创建的仓库中获取快照列表或删除快照。