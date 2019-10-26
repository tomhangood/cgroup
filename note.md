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

**NOTE:** subsystem 实际上就是 cgroups 的资源控制系统，每种 subsystem 独立地控制一种资源</br>


#### cgroups 实现方式及工作原理简介

##### cgroups 实现结构讲解

![Alt text](/pic/cgroup1.png)</br>
**图 5 cgroups 相关结构体一览**</br>
**NOTE：** 一个 task 只对应一个css_set结构，但是一个css_set可以被多个 task 使用.</br>

**task**:
1. 一个是css_set * cgroups，表示指向css_set（包含进程相关的 cgroups 信息）的指针.
2. list_head cg_list，是一个链表的头指针，这个链表包含了所有的链到同一个css_set的 task 进程.</br>

**css_set**:
每个css_set结构中都包含了一个指向cgroup_subsys_state（**包含进程与一个特定子系统相关的信息**）的指针数组。cgroup_subsys_state则指向了cgroup结构（包含一个 cgroup 的所有信息），通过这种方式间接的把一个进程和 cgroup 联系了起来，如下图 6。</br>
**task ----> cgroup**</br>
![Alt text](/pic/cgroup2.png)</br>
**图 6 从 task 结构开始找到 cgroup 结构**</br>

cgroup结构体中有一个list_head css_sets字段，它是一个头指针，指向由cg_cgroup_link（包含 cgroup 与 task 之间多对多关系的信息，后文还会再解释）形成的链表。由此获得的每一个cg_cgroup_link都包含了一个指向css_set * cg字段，指向了每一个 task 的css_set。css_set结构中则包含tasks头指针，指向所有链到此css_set的 task 进程构成的链表。至此，我们就明白如何查看在同一个 cgroup 中的 task 有哪些了，如下图
**cgroup ----> task**</br>
![Alt text](/pic/cgroup3.png)</br>
**图 7 cglink 多对多双向查询**</br>

css_set中有两种方式定位到cgroup：
1. css->cgroup_subsys_state->cgroup
2. css->cg_cgroup_link->cgroup</br>
**Q**:
but, why there are two ways to locate the cgroup?</br>
**Because**:task 与 cgroup 之间是多对多的关系.</br>
Moreover,如果两张表是多对多的关系，那么如果不加入第三张关系表，就必须为一个字段的不同添加许多行记录，导致大量冗余。通过从主表和副表各拿一个主键新建一张关系表，可以提高数据查询的灵活性和效率。<br>

一个 task 可能处于不同的 cgroup，只要这些 cgroup 在不同的 hierarchy 中，并且每个 hierarchy 挂载的子系统不同；另一方面，一个 cgroup 中可以有多个 task，这是显而易见的，但是这些 task 因为可能还存在在别的 cgroup 中，所以它们对应的css_set也不尽相同，所以一个 cgroup 也可以对应多个·css_set。</br>

**NOTE:** 在系统运行之初，内核的主函数就会对root cgroups和css_set进行初始化，每次 task 进行 fork/exit 时，**都会附加（attach）/ 分离（detach）对应的css_set**。</br>

**NOTE:** 添加cg_cgroup_link主要是出于**性能方面**的考虑，一是节省了task_struct结构体占用的内存，二是提升了进程fork()/exit()的速度。</br>

![Alt text](/pic/cgroup4.png)</br>
**图 8 css_set 与 hashtable 关系**
当 task 从一个 cgroup 中移动到另一个时，它会得到一个新的css_set指针。如果所要加入的 cgroup 与现有的 cgroup 子系统相同，那么就重复使用现有的css_set，否则就分配一个新css_set。所有的css_set通过一个哈希表进行存放和查询，如上图 8 中所示，hlist_node hlist就指向了css_set_table这个 hash 表。</br>


定义子系统的结构体是cgroup_subsys，在图 9 中可以看到，cgroup_subsys中定义了一组函数的接口，让各个子系统自己去实现，类似的思想还被用在了cgroup_subsys_state中，cgroup_subsys_state并没有定义控制信息，只是定义了各个子系统都需要用到的公共信息，由各个子系统各自按需去定义自己的控制信息结构体，最终在自定义的结构体中把cgroup_subsys_state包含进去，然后内核通过container_of（这个宏可以通过一个结构体的成员找到结构体自身）等宏定义来获取对应的结构体。</br>
![Alt text](/pic/cgroup5.png)</br>
**图 9 cgroup 子系统结构体**</br>

对于container_of这部分，以下面的freezer为例：</br>
```
struct freezer {
	struct cgroup_subsys_state	css;
	unsigned int			state;
};

static DEFINE_MUTEX(freezer_mutex);

static inline struct freezer *css_freezer(struct cgroup_subsys_state *css)
{
	return css ? container_of(css, struct freezer, css) : NULL;
}

```

#### 基于 cgroups 实现结构的用户层体现
不多说了，下面这篇博客写的太棒了，直接copy过来：</br>
https://www.infoq.cn/article/docker-kernel-knowledge-cgroups-resource-isolation </br>
了解了 cgroups 实现的代码结构以后，再来看用户层在使用 cgroups 时的限制，会更加清晰。

在实际的使用过程中，你需要通过挂载（mount）cgroup文件系统新建一个层级结构，挂载时指定要绑定的子系统，缺省情况下默认绑定系统所有子系统。把 cgroup 文件系统挂载（mount）上以后，你就可以像操作文件一样对 cgroups 的 hierarchy 层级进行浏览和操作管理（包括权限管理、子文件管理等等）。除了 cgroup 文件系统以外，内核没有为 cgroups 的访问和操作添加任何系统调用。

如果新建的层级结构要绑定的子系统与目前已经存在的层级结构完全相同，那么新的挂载会重用原来已经存在的那一套（指向相同的 css_set）。否则如果要绑定的子系统已经被别的层级绑定，就会返回挂载失败的错误。如果一切顺利，挂载完成后层级就被激活并与相应子系统关联起来，可以开始使用了。

目前无法将一个新的子系统绑定到激活的层级上，或者从一个激活的层级中解除某个子系统的绑定。

当一个顶层的 cgroup 文件系统被卸载（umount）时，如果其中创建后代 cgroup 目录，那么就算上层的 cgroup 被卸载了，层级也是激活状态，其后代 cgoup 中的配置依旧有效。只有递归式的卸载层级中的所有 cgoup，那个层级才会被真正删除。

层级激活后，/proc目录下的每个 task PID 文件夹下都会新添加一个名为cgroup的文件，列出 task 所在的层级，对其进行控制的子系统及对应 cgroup 文件系统的路径。

一个 cgroup 创建完成，不管绑定了何种子系统，其目录下都会生成以下几个文件，用来描述 cgroup 的相应信息。同样，把相应信息写入这些配置文件就可以生效，内容如下。

tasks：这个文件中罗列了所有在该 cgroup 中 task 的 PID。该文件并不保证 task 的 PID 有序，把一个 task 的 PID 写到这个文件中就意味着把这个 task 加入这个 cgroup 中。
cgroup.procs：这个文件罗列所有在该 cgroup 中的线程组 ID。该文件并不保证线程组 ID 有序和无重复。写一个线程组 ID 到这个文件就意味着把这个组中所有的线程加到这个 cgroup 中。
notify_on_release：填 0 或 1，表示是否在 cgroup 中最后一个 task 退出时通知运行release agent，默认情况下是 0，表示不运行。
release_agent：指定 release agent 执行脚本的文件路径（该文件在最顶层 cgroup 目录中存在），在这个脚本通常用于自动化umount无用的 cgroup。
除了上述几个通用的文件以外，绑定特定子系统的目录下也会有其他的文件进行子系统的参数配置。

在创建的 hierarchy 中创建文件夹，就类似于 fork 中一个后代 cgroup，后代 cgroup 中默认继承原有 cgroup 中的配置属性，但是你可以根据需求对配置参数进行调整。这样就把一个大的 cgroup 系统分割成一个个嵌套的、可动态变化的“软分区”。
