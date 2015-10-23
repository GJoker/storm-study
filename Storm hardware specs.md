###Zookeeper

 - 推荐使用专用机器，因为ZK是Storm的瓶颈所在。
  - 每台机器开启一个ZK实例
  - 在一些情况下可以使用虚拟机，需要注意的是其他的虚拟机或者程序在Storm主机上运行时，会影响到ZK的性能，尤其是当它们还执行I/O操作时
 - I/O是Zookeeper的瓶颈
  - 将ZK的存储放在它自己的磁盘设备上。
  - 固态硬盘大大地提高了性能。
  - 通常情况下，每写入数据ZK都会同步到磁盘上， 这将进行两次操作（1次是同步数据，1次是同步数据的日志）。当所有的工作进程都发送心跳到ZK集群时，同步的工作量将会大大地增加。
  - 在ZK节点上监测I/O录入。
 - 在生产环境中推荐同时启动大于或者等于3台的机器作为ZK集群，这样当有一台Zk服务器失败时，整个集群还能正常运行。