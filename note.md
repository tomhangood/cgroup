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
3. subsystem（子系统）：cgroups 中的 subsystem 就是一个**资源调度控制器**（Resource Controller）。比如 CPU 子系统可以控制 CPU 时间分配，内存子系统可以限制 cgroup 内存使用量。
4. hierarchy（层级树）：hierarchy 由**一系列 cgroup** 以一个**树状**结构排列而成，每个 hierarchy 通过**绑定**对应的 **subsystem** 进行资源调度。hierarchy 中的 cgroup 节点可以包含零或多个子节点，子节点继承父节点的属性。整个系统可以有多个 hierarchy。</br>
**NOTE:** cgroups 的模型则是由多个 hierarchy 构成的森林</br>
Why?</br>
Because:如果只有一个 hierarchy，那么所有的 task 都要受到绑定其上的 subsystem 的限制，会给那些不需要这些限制的 task 造成麻烦。</br>


##### 重要规则：
**规则 1**： 同一个 hierarchy 可以附加一个或多个 subsystem。如下图 1，cpu 和 memory 的 subsystem 附加到了一个 hierarchy。</br>
![Alt text](/pic/pic_1.png)</br>
**图 1 同一个 hierarchy 可以附加一个或多个 subsystem**</br>

**规则 2**： 一个 subsystem 可以附加到多个 hierarchy，当且仅当这些 hierarchy 只有这唯一一个 subsystem。如下图 2，小圈中的数字表示 subsystem 附加的时间顺序，CPU subsystem 附加到 hierarchy A 的同时不能再附加到 hierarchy B，因为 hierarchy B 已经附加了 memory subsystem。如果 hierarchy B 与 hierarchy A 状态相同，没有附加过 memory subsystem，那么 CPU subsystem 同时附加到两个
 hierarchy 是可以的。</br>
![Alt text](/pic/pic_2.png)</br>
**图 2 一个已经附加在某个 hierarchy 上的 subsystem 不能附加到其他含有别的 subsystem 的 hierarchy 上**</br>

**规则 3**： 系统每次新建一个 hierarchy 时，该系统上的**所有 task 默认构成了这个新建的 hierarchy 的初始化 cgroup**，这个 cgroup 也称为 **root cgroup**。对于你创建的每个 hierarchy，task 只能存在于其中一个 cgroup 中，即**一个 task 不能存在于同一个 hierarchy 的不同 cgroup 中**，但是一个 task 可以存在在不同 hierarchy 中的多个 cgroup 中。如果操作时把一个 task 添加到同一个 hierarchy 中的另一个 cgroup 中，则会从第一个 cgroup 中移除。在下图 3 中可以看到，httpd进程已经加入到 hierarchy A 中的/cg1而不能加入同一个 hierarchy 中的/cg2，但是可以加入 hierarchy B 中的/cg3。实际上不允许加入同一个 hierarchy 中的其他 cgroup 野生,**为了防止出现矛盾**，如 CPU subsystem 为/cg1分配了 30%，而为/cg2分配了 50%，此时如果httpd在这两个 cgroup 中，就会出现矛盾。</br>
![Alt text](/pic/pic_3.png)</br>
**图 3 一个 task 不能属于同一个 hierarchy 的不同 cgroup**</br>


**规则 4**： 进程（task）在 fork 自身时创建的子任务（child task）默认与原 task 在同一个 cgroup 中，但是 child task 允许被移动到不同的 cgroup 中。即 fork 完成后，父子进程间是完全独立的。如下图 4 中，小圈中的数字表示 task 出现的时间顺序，当httpd刚 fork 出另一个httpd时，在同一个 hierarchy 中的同一个 cgroup 中。但是随后如果 PID 为 4840 的httpd需要移动到其他 cgroup 也是可以的，因为父子任务间已经独立。总结起来就是：初始化时子任务与父任务在同一个 cgroup，但是这种关系随后可以改变。</br>
![Alt text](/pic/pic_4.png)</br>
**图 4 刚 fork 出的子进程在初始状态与其父进程处于同一个 cgroup**</br>
