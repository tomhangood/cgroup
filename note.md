### 用于记录目前看到的Key Point!

对开发者来说，cgroups 有如下四个有趣的特点：

1. cgroups 的 API 以一个**伪文件系统**的方式实现，即用户可以通过文件操作实现 cgroups 的组织管理。</br>
2. cgroups 的组织管理操作单元可以细粒度到线程级别，**用户态代码也可以针对系统分配的资源创建和销毁 cgroups**, 从而实现资源再分配和管理。</br>
3. 所有资源管理的功能都以“subsystem（子系统）”的方式实现，**接口统一**。</br>
4. 子进程创建之初与其父进程处于**同一个 cgroups 的控制组**。</br>


本质上来说，cgroups 是内核附加在程序上的**一系列钩子**（hooks），通过程序运行时对资源的调度触发相应的钩子以达到资源追踪和限制的目的。</br>

实现 cgroups 的主要目的是为不同用户层面的资源管理，提供一个**统一化的接口。** </br>

##### cgroups 的作用

1. 资源限制（Resource Limitation）：cgroups 可以对进程组使用的资源总额进行限制。如设定应用运行时使用内存的上限，一旦超过这个配额就发出 OOM（Out of Memory）。
2. 优先级分配（Prioritization）：通过分配的 CPU 时间片数量及硬盘 IO 带宽大小，实际上就相当于控制了进程运行的优先级。
3. 资源统计（Accounting）： cgroups 可以统计系统的资源使用量，如 CPU 使用时长、内存用量等等，这个功能非常适用于计费。
4. 进程控制（Control）：cgroups 可以对进程组执行挂起、恢复等操作。

**NOTE:** 以上四条非常非常重要，洗完可以对照着下表去体会:</br>
![Alt text](/pic/cgroup.png)</br>

##### 术语表

1. task（任务）：cgroups 的术语中，task 就表示系统的一个进程。
2. cgroup（控制组）：cgroups 中的**资源控制**都以**cgroup 为单位**实现。cgroup 表示按某种资源控制标准划分而成的任务组，包含一个或多个子系统。一个任务可以加入某个 cgroup，也可以从某个 cgroup 迁移到另外一个 cgroup。
3. subsystem（子系统）：cgroups 中的 subsystem 就是一个资源调度控制器（Resource Controller）。比如 CPU 子系统可以控制 CPU 时间分配，内存子系统可以限制 cgroup 内存使用量。
4. hierarchy（层级树）：hierarchy 由一系列 cgroup 以一个树状结构排列而成，每个 hierarchy 通过绑定对应的 subsystem 进行资源调度。hierarchy 中的 cgroup 节点可以包含零或多个子节点，子节点继承父节点的属性。整个系统可以有多个 hierarchy。
