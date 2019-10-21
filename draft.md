#### Basic concept: ####

A **cgroup** associates a set of tasks with a set of parameters for one
or more subsystems.

**"**A **subsystem** is a module that makes use of the task grouping
facilities provided by cgroups to treat groups of tasks in
particular ways.**"** A subsystem is typically a "resource controller" that
schedules a resource or applies per-cgroup limits, but it may be
anything that wants to act on a group of processes, e.g. a
virtualization subsystem.

A **hierarchy** is a set of cgroups arranged in a tree, such that
every task in the system is in exactly one of the cgroups in the
hierarchy, and a set of subsystems; each subsystem has system-specific
state attached to each cgroup in the hierarchy.  Each hierarchy has
an instance of the cgroup virtual filesystem associated with it.


按照资源的划分，系统被划分成了不同的子系统(subsystem)，正如我们上面列出的cpu, cpuset, blkio...每种资源独立构成一个subsystem.

可以将cgroup的架构抽象的理解为多根的树结构，一个hierarchy代表一棵树，树上绑定一个或多个subsystem.而树的叶子则是cgroup,一个cgroup具体的限制了某种资源。一个或多个cgroup组成一个css_set。简单来讲，就是一个资源限制集合(css_set)对一种subsystem(cpu，devices)的限制条件只能有一个，这是显然的吧...最终的task(进程)同css_set关联，从而达到限制资源的目的。具体cgroup和css_set 关联的方式, **see the second chart**


![Alt text](/pic/1.png)

#### cgroup how to create: ####

/cgroup
```
cgroup_init
  err = register_filesystem(&cgroup_fs_type);

static struct file_system_type cgroup_fs_type = {
        .name = "cgroup",
        .mount = cgroup_mount,
        .kill_sb = cgroup_kill_sb,
};

```

Create a directory in cgroup, like /cgroup/xxxx
```
cgroup_mount
  ret = cgroup_setup_root(root, opts.subsys_mask);

cgroup_setup_root
  root->kf_root = kernfs_create_root(&cgroup_kf_syscall_ops,KERNFS_ROOT_CREATE_DEACTIVATED,root_cgrp)

static struct kernfs_syscall_ops cgroup_kf_syscall_ops = {
        .remount_fs             = cgroup_remount,
        .show_options           = cgroup_show_options,
        .mkdir                  = cgroup_mkdir,
        .rmdir                  = cgroup_rmdir,
        .rename                 = cgroup_rename,
};


cgroup_mkdir()
{



}
```

**the key flow chart**

![Alt text](/pic/cgroup.png)
