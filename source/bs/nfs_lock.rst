nfs lock 1670650
================

在单台机器上nfs lock 不成功

安装nfs

::

   #安装软件
   yum install nfs-utils
   #关闭防火墙
   systemctl stop firewalld.service
   #创建挂载目录
   mkdir nfs-test-dir
   #配置共享文件夹
   cat /etc/exports
   /root/nfs-test-dir *(rw,sync,no_root_squash)

挂载

::

   mount -t nfs -o vers=3 localhost:/root/nfs-test-dir /tmp

执行测试

.. code:: shell-session

   [root@redhat75 ~]# ./test.sh /tmp

   Test locking file: /tmp/flock-user-test.7901
   flock: 10: Bad file descriptor
   Lock failed: 65
   flock: 10: Bad file descriptor
   Unlock failed: 65

脚本内容：

.. code:: bash

   #!/bin/bash
   # Usage: $0 <directory> [<directory>...]

   for directory in $@
   do
       test_file="$directory/flock-user-test.$$"
       printf "\nTest locking file: %s\n" "$test_file"
       touch "$test_file"
       {
           #fuser "$test_file"
           flock -w 2 -x $test_fd || printf "Lock failed: %d\n" $?
           #ls -l /proc/$$/fd
           #fuser "$test_file"
           flock -u -x $test_fd || printf "Unlock failed: %d\n" $?
       } {test_fd}<"$test_file"
       rm -f "$test_file"
   done

注意：上述验证，在单台设备上完成，否则可能不会重现。
本人使用redhat7.6作为服务端，使用ubuntu18.04作为客户端挂在nfs时，是没有出现这个问题的。

经过验证, ARM平台上redhat7.4 ok, 7.5 no, 7.6 no, 8.0 ok，X86平台上 7.6
ok，如下表

======= ======= ======= ======= ========
架构    版本                   
======= ======= ======= ======= ========
\       RHEL7.4 RHEL7.5 RHEL7.6 RHEL 8.0
X86     -       -       ok      -
aarch64 ok      fail    fail    ok
======= ======= ======= ======= ========

各redhat版本对应的内核版本如下

::

   RHEL7.4  4.11.0-44.el7a.aarch64
   RHEL7.5  4.14.0-49.el7a.aarch64
   RHEL7.6  4.14.0-115.el7a.aarch64
   RHEL8.0  4.18.0-64.el8.aarch64

strace 跟踪
-----------

::

   strace -ff -o strace-thread-output ./test.sh /tmp

是在执行flock系统调用的时候失败，主要差异是：

::

   RHEL7.6 flock(10,LOCK_EX)   = -1 EBADF(Bad file descriptor)
   RHEL7.4 flock(10,LOCK_EX)   = 0

flock实现
---------

在文件 ``fs/locks.c`` 把LOCK_EX翻译成F_WRLCK

.. code:: c

   static inline int flock_translate_cmd(int cmd) {
           if (cmd & LOCK_MAND)
                   return cmd & (LOCK_MAND | LOCK_RW);
           switch (cmd) {
           case LOCK_SH:
                   return F_RDLCK;
           case LOCK_EX:
                   return F_WRLCK;
           case LOCK_UN:
                   return F_UNLCK;
           }
           return -EINVAL;
   }

在文件 ``fs/locks.c`` 定义了系统调用flock,由f_op结构体中的函数指针指示

.. code:: c

   SYSCALL_DEFINE2(flock, unsigned int, fd, unsigned int, cmd)
   {
       /*......*/
       
       if (f.file->f_op->flock && is_remote_lock(f.file))
                   error = f.file->f_op->flock(f.file,
                                             (can_sleep) ? F_SETLKW : F_SETLK,
                                             lock);
       /*.....*/
   }

在文件\ ``fs/nfs/file.c`` 指示了对NFS文件的lock操作是nfs_lock

.. code:: c

   const struct file_operations nfs_file_operations = {
           .llseek         = nfs_file_llseek,
           .read_iter      = nfs_file_read,
           .write_iter     = nfs_file_write,
           .mmap           = nfs_file_mmap,
           .open           = nfs_file_open,
           .flush          = nfs_file_flush,
           .release        = nfs_file_release,
           .fsync          = nfs_file_fsync,
           .lock           = nfs_lock,
           .flock          = nfs_flock, //这里
           .splice_read    = generic_file_splice_read,
           .splice_write   = iter_file_splice_write,
           .check_flags    = nfs_check_flags,
           .setlease       = simple_nosetlease,
   };
   EXPORT_SYMBOL_GPL(nfs_file_operations);

| 但是不同redhat版本的内核对nfs_flock的实现不一样： 
| **RHEL7.4 4.11.0-44.el7a.aarch64 kernel-alt-4.11.0-44.el7a**

.. code:: c

   int nfs_flock(struct file *filp, int cmd, struct file_lock *fl)
   {
           struct inode *inode = filp->f_mapping->host;
           int is_local = 0;

           dprintk("NFS: flock(%pD2, t=%x, fl=%x)\n",
                           filp, fl->fl_type, fl->fl_flags);

           if (!(fl->fl_flags & FL_FLOCK))
                   return -ENOLCK;

           /*
            * The NFSv4 protocol doesn't support LOCK_MAND, which is not part of
            * any standard. In principle we might be able to support LOCK_MAND
            * on NFSv2/3 since NLMv3/4 support DOS share modes, but for now the
            * NFS code is not set up for it.
            */
           if (fl->fl_type & LOCK_MAND)
                   return -EINVAL;

           if (NFS_SERVER(inode)->flags & NFS_MOUNT_LOCAL_FLOCK)
                   is_local = 1;

           /* We're simulating flock() locks using posix locks on the server */
           if (fl->fl_type == F_UNLCK)
                   return do_unlk(filp, cmd, fl, is_local);
           return do_setlk(filp, cmd, fl, is_local);
   }
   EXPORT_SYMBOL_GPL(nfs_flock);

**RHEL7.5 4.14.0-49.el7a.aarch64 kernel-alt-4.14.0-49.el7a**

.. code:: c

   int nfs_flock(struct file *filp, int cmd, struct file_lock *fl)
   {
           struct inode *inode = filp->f_mapping->host;
           int is_local = 0;

           dprintk("NFS: flock(%pD2, t=%x, fl=%x)\n",
                           filp, fl->fl_type, fl->fl_flags);

           if (!(fl->fl_flags & FL_FLOCK))
                   return -ENOLCK;

           /*
            * The NFSv4 protocol doesn't support LOCK_MAND, which is not part of
            * any standard. In principle we might be able to support LOCK_MAND
            * on NFSv2/3 since NLMv3/4 support DOS share modes, but for now the
            * NFS code is not set up for it.
            */
           if (fl->fl_type & LOCK_MAND)
                   return -EINVAL;

           if (NFS_SERVER(inode)->flags & NFS_MOUNT_LOCAL_FLOCK)
                   is_local = 1;

           /*
            * VFS doesn't require the open mode to match a flock() lock's type.
            * NFS, however, may simulate flock() locking with posix locking which
            * requires the open mode to match the lock type.
            */
           switch (fl->fl_type) {
           case F_UNLCK:
                   return do_unlk(filp, cmd, fl, is_local);
           case F_RDLCK:
                   if (!(filp->f_mode & FMODE_READ))
                           return -EBADF;
                   break;
           case F_WRLCK:
                   if (!(filp->f_mode & FMODE_WRITE))
                           return -EBADF;
           }

           return do_setlk(filp, cmd, fl, is_local);
   }
   EXPORT_SYMBOL_GPL(nfs_flock);

**RHEL8.0 4.18.0-64.el8.aarch64 kernel-4.18.0-64.el8**

.. code:: c

   int nfs_flock(struct file *filp, int cmd, struct file_lock *fl)
   {
           struct inode *inode = filp->f_mapping->host;
           int is_local = 0;

           dprintk("NFS: flock(%pD2, t=%x, fl=%x)\n",
                           filp, fl->fl_type, fl->fl_flags);

           if (!(fl->fl_flags & FL_FLOCK))
                   return -ENOLCK;

           /*
            * The NFSv4 protocol doesn't support LOCK_MAND, which is not part of
            * any standard. In principle we might be able to support LOCK_MAND
            * on NFSv2/3 since NLMv3/4 support DOS share modes, but for now the
            * NFS code is not set up for it.
            */
           if (fl->fl_type & LOCK_MAND)
                   return -EINVAL;

           if (NFS_SERVER(inode)->flags & NFS_MOUNT_LOCAL_FLOCK)
                   is_local = 1;

           /* We're simulating flock() locks using posix locks on the server */
           if (fl->fl_type == F_UNLCK)
                   return do_unlk(filp, cmd, fl, is_local);
           return do_setlk(filp, cmd, fl, is_local);
   }
   EXPORT_SYMBOL_GPL(nfs_flock);

找到nfs_flock的更改历史
-----------------------

查看关于这个文件的更改

::

   git log --oneline kernel-alt-4.11.0-44.el7a..kernel-4.18.0-64.el8 -- fs/nfs/file.c

结果如下

.. code-block:: console

   fcfa447 NFS: Revert "NFS: Move the flock open mode check into nfs_flock()"
   bf4b490 NFS: various changes relating to reporting IO errors.
   e973b1a5 NFS: Sync the correct byte range during synchronous writes
   779eafa NFS: flush data when locking a file to ensure cache coherence for mmap.
   6ba80d4 NFS: Optimize fallocate by refreshing mapping when needed.
   442ce04 NFS: invalidate file size when taking a lock.
   c373fff NFSv4: Don't special case "launder"
   f30cb75 NFS: Always wait for I/O completion before unlock
   e129372 NFS: Move the flock open mode check into nfs_flock()

锁定两个commit：

::

   e129372 NFS: Move the flock open mode check into nfs_flock()
   fcfa447 NFS: Revert "NFS: Move the flock open mode check into nfs_flock()"

再分析两个commit出现的位置，在7.4之后，commit e129372 代码进行了修改，
到8.0版本之前又恢复了回来，和测试现象一致， 7.4和8.0执行OK，
但是7.5和7.6有问题。

::

   RHEL8.0  4.18.0-64.el8.aarch64      kernel-4.18.0-64.el8

   fcfa447 NFS: Revert "NFS: Move the flock open mode check into nfs_flock()"

   RHEL7.6  4.14.0-115.el7a.aarch64    kernel-alt-4.14.0-115.el7a   
   RHEL7.5  4.14.0-49.el7a.aarch64     kernel-alt-4.14.0-49.el7a

   e129372 NFS: Move the flock open mode check into nfs_flock()

   RHEL7.4  4.11.0-44.el7a.aarch64     kernel-alt-4.11.0-44.el7a

根因:文件打开模式和锁类型不应该强制要求一样
-------------------------------------------

根因是\ ``commit e129372``\ 引入了这个问题。

::

   diff --git a/fs/nfs/file.c b/fs/nfs/file.c
   index 6682139..b7f4af3 100644
   --- a/fs/nfs/file.c
   +++ b/fs/nfs/file.c
   @@ -820,9 +820,23 @@ int nfs_flock(struct file *filp, int cmd, struct file_lock *fl)
           if (NFS_SERVER(inode)->flags & NFS_MOUNT_LOCAL_FLOCK)
                   is_local = 1;

   -       /* We're simulating flock() locks using posix locks on the server */
   -       if (fl->fl_type == F_UNLCK)
   +       /*
   +        * VFS doesn't require the open mode to match a flock() lock's type.
   +        * NFS, however, may simulate flock() locking with posix locking which
   +        * requires the open mode to match the lock type.
   +        */
   +       switch (fl->fl_type) {
   +       case F_UNLCK:
                   return do_unlk(filp, cmd, fl, is_local);
   +       case F_RDLCK:
   +               if (!(filp->f_mode & FMODE_READ))
   +                       return -EBADF;
   +               break;
   +       case F_WRLCK:
   +               if (!(filp->f_mode & FMODE_WRITE))
   +                       return -EBADF;
   +       }
   +
           return do_setlk(filp, cmd, fl, is_local);
    }
    EXPORT_SYMBOL_GPL(nfs_flock);

在执行\ ``flock(10,LOCK_EX)``\ 时，LOCK_EX会翻译成写锁F_WRLCK，但是文件打开模式可以使用strace观察到是只读模式O_RDONLY打开的，复制了一份文件

.. code:: c

   openat(AT_FDCWD, "/tmp//flock-user-test.4600", O_RDONLY) = 3
   fcntl(3, F_DUPFD, 10)                   = 10
   ····
   flock(10,LOCK_EX)

所以代码进入

.. code:: c

   +       case F_WRLCK:
   +               if (!(filp->f_mode & FMODE_WRITE))
   +                       return -EBADF;
   +       }

| 要求，当文件申请写锁时，文件的打开模式f_mode必须是FMODE_WRITE，但是f_mode此时是FMODE_READ,
  **所以出错**\ 。
| 关于f_mode的描述，在文件\ ``/inlcude/linux/fs.h``\ 中有描述

.. code:: c

   /*
    * flags in file.f_mode.  Note that FMODE_READ and FMODE_WRITE must correspond
    * to O_WRONLY and O_RDWR via the strange trick in __dentry_open()
    */

关于commit
revert的原因，可以查看\ ``commit fcfa447``,作者写到，在NFS的实现中，会用posix的locking机制模拟flock，不

.. code-block:: console

   commit fcfa447062b2061e11f68b846d61cbfe60d0d604
   Author: Benjamin Coddington <bcodding@redhat.com>
   Date:   Fri Nov 10 06:27:49 2017 -0500

       NFS: Revert "NFS: Move the flock open mode check into nfs_flock()"

       Commit e12937279c8b "NFS: Move the flock open mode check into nfs_flock()"
       changed NFSv3 behavior for flock() such that the open mode must match the
       lock type, however that requirement shouldn't be enforced for flock().

       Signed-off-by: Benjamin Coddington <bcodding@redhat.com>
       Cc: stable@vger.kernel.org # v4.12
       Signed-off-by: Anna Schumaker <Anna.Schumaker@Netapp.com>

编译新内核验证
--------------

在内核代码根目录，切换到redhat7.6的内核代码

::

   git checkout -b fix_flock kernel-alt-4.14.0-115.el7a

合入commit fcfa447

::

   git cherry-pick fcfa447

添加内核编译配置文件

::

   cp ~/config-4.14.0-115.el7a.aarch64 .config

执行编译脚本

::

   build-kernel-natively.sh

复制内核文件到目标主机

::

   scp /root/rpmbuild/RPMS/aarch64/kernel-4.14.0_alt_115_fix_nfs_flock_2019_02_01-3.aarch64.rpm me@192.168.1.126:~/

执行安装并重启

::

   yum install kernel-4.14.0_alt_115_fix_nfs_flock_2019_02_01-3.aarch64.rpm
   reboot -f

选择新内核启动系统

::



         Red Hat Enterprise Linux Server (4.14.0-alt-115-fix-nfs-flock-2019-02-01>
         Red Hat Enterprise Linux Server (4.14.0-115.el7a.aarch64) 7.6 (Maipo)
         Red Hat Enterprise Linux Server (0-rescue-f0b91f9908f44910b44f62583e5da5>














         Use the ^ and v keys to change the selection.
         Press 'e' to edit the selected item, or 'c' for a command prompt.

确认版本更新

::

   [root@redhat76 ~]# uname -a
   Linux redhat76 4.14.0-alt-115-fix-nfs-flock-2019-02-01 #3 SMP Fri Feb 1 15:47:15 CST 2019 aarch64 aarch64 aarch64 GNU/Linux
   [root@redhat76 ~]# cat /etc/redhat-release
   Red Hat Enterprise Linux Server release 7.6 (Maipo)

重新验证成功

::

   [root@redhat76 ~]#bash -x ./test.sh /tmp
   + for directory in '$@'
   + test_file=/tmp/flock-user-test.9916
   + printf '\nTest locking file: %s\n' /tmp/flock-user-test.9916

   Test locking file: /tmp/flock-user-test.9916
   + touch /tmp/flock-user-test.9916
   + flock -w 2 -x 10
   + flock -u -x 10
   + rm -f /tmp/flock-user-test.9916
