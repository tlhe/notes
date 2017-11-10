#Elasticsearch停机

有序的关闭Elasticsearch来确保Elasticsearch有机会清理和关闭未完成得资源。譬如：节点关闭后有序的从集群中移除、同步传输日志到磁盘以及一些其他的相关清理活动。你可以确保Elasticsearch有序的停机来帮助Elasticsearch正确的停止。

如果Elasticsearch作为一个服务运行，你可以通过你安装的服务管理功能来停止Elasticsearch。

如果你是在控制台直接运行的Elasticsearch，你可以通过发送conrtol + C来停止，或者是在POSIX系统发送SIGTERM信号给Elasticsearch进程。你可以通过各种各样的工具获取PID来发送信号（如：ps或jps）：
```
$ jps | grep Elasticsearch
14542 Elasticsearch
```
通过启动日志：
```
[2016-07-07 12:26:18,908][INFO ][node] [I8hydUG] version[5.0.0-alpha4], pid[15399], build[3f5b994/2016-06-27T16:23:46.861Z], OS[Mac OS X/10.11.5/x86_64], JVM[Oracle Corporation/Java HotSpot(TM) 64-Bit Server VM/1.8.0_92/25.92-b14]
```
或者通过启动时指定的PID文件获取：
```
$ ./bin/elasticsearch -p /tmp/elasticsearch-pid -d
$ cat /tmp/elasticsearch-pid && echo
15516
$ kill -SIGTERM 15516
```
#致命错误停机
在Elasticsearch虚拟机运行期间，可能出现某些致命错误把虚拟机标记为可疑状态。这些致命错误可能包含虚拟机内部错误、严重的I/O错误。

当Elasticsearch检测到虚拟机遇到这样一个致命错误时，Elasticsearch将尝试记录错误，然后将停止虚拟机。当Elasticsearch发起一个这样的关闭时，它没有经过上述的有序关闭。Elasticsearch将会返回一个特定的状态码来标识这个错误。
|错误原因|	错误码|
|-|-|
|JVM内部错误|128|
|内存溢出|127|
|堆溢出|126|
|未知虚拟机错误|125|
|严重I/O错误	|124|
|未知致命错误	|1|