容器技术是 云发展的一个重要基础，docker就是当前很火的一种容器技术。

之前就知道docker利用了linux的cgruop, namespace + chroot + 联合文件系统实现的。

本章力求从源码角度对docker进行分析, docker版本为：https://github.com/moby/moby/tree/v19.03.9

章节安排：

（1）了解linux namespaces, cgroup, choot，联合文件系统的原理

（2）了解docker源码结构

（3）以常见的docker run nginx ls命令为主线,  从源码入手了解该命令背后的详细过程
