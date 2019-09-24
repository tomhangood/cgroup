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

![Alt text](/pic/cgroup.pdf)
