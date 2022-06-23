---
title: Linux开启大页内存
description:
authors:
  - Ray
tags:
categories:
  - 数据库
series:
date: "2022-01-24"
featuredImage: featured-image.webp
summary: 开启大页内存过程
---

> 大页内存是集成在 Linux 2.6 内核的一个特性。与传统的 4K 页面相比，启用大页内存可以使操作系统支持更大的页面。使用大页内存将会减少系统维护 page table entries(页表条目，PTE)所消耗的资源，从而提高系统性能。大页内存对于 32 位和 64 位的配置都有用。大页内存的大小从 2MB 到 256MB 不等，依赖于内核版本和硬件架构。对于 Oracle 数据库，使用大页内存可以减少操作系统对页面状态的维护，并且能够提升 Translation Lookaside Buffer (页面缓冲，TLB)的命中率。

_注：透明大页目前不是替代手工配置大页内存的替代方案。_

### 检查大页内存的分配

在你的操作系统中检查是否启用了大页内存。

在 Linux 平台上，Oracle 推荐在数据库服务器上启用大页内存来获取更好的性能。在启用了大页内存的服务器上升级 GI 和 Database 时，Oracle 建议检查大页内存分配需求。

GIMR 和大页内存：Oracle GI 安装的时候会包括 Grid Infrastructure Management Repository (GIMR)。当集群的成员节点上配置了大页内存时，GIMR 的 SGA 会被分配到大页内存区域，最多将会占用 1GB 大小。GI 将会在集群安装 Database 之前启动。

如果集群成员节点的内存大页分配与该节点所有 DB 实例的 SGA 存在冲突，则需要找出占用普通页面的 SGA，用大页内存来代替它，否则将会影响性能。为了避免发生这样的问题，当计划升级之前，需要确保保留的大页内存足够大来满足内存需求。

预留的大页内存应当大于所有 DB 实例的 SGA 的总和以及 GIMR 的 SGA 部分。

### 在 Linux 上使用大页内存

为了使 Oracle 数据库在 Linux 系统上能够使用大页内存，需要设置 vm.nr_hugepages 内核参数，将它设置为所需要保留的大页内存的值。一定要分配足够的大页来承载数据库实例的 SGA。可以将所有实例的 SGA 大小的和除以一个大页面的大小，然后向上取整，确定所需参数值。

可以用下面的方法来查看默认的大页大小

```console
# grep Hugepagesize /proc/meminfo
```

例如，如果/proc/meminfo 显示的大页大小为 2M，所有实例的 SGA 加在一起的大小是 1.6G，则 vm.nr_hugepages 内核参数的值应当设置为 820(1.6GB/2M=819.2)

### 利用大页内存调优 SGA

在没有大页内存的情况下，Linux 保持内存页面为 4KB。当分配页面给数据库的 SGA，在页面的生命周期内(脏块、空闲、映射地址空间等)，操作系统必须为每一个 4K 页面持续地更新页表。

配置了大页内存，操作系统的页表(虚拟内存到物理内存大映射)会变得更小，因为每一个页表项将会指向从 2M 到 256M 不等的页面。

同样，内核将会监控更少的页面及其生命周期。例如，如果你在 64 位的硬件上使用大页内存，你打算映射 256MB 内存，你将需要一个页表项(PTE)。如果没有使用大页内存，同样是映射 256MB 内存，那么你需要 256MB \* 1024KB 4KB = 65536 个页表项(PTE)。

大页内存有如下好处：

> 通过增加 TLB 命中率提升性能。
> 分配给 SGA 的大页内存在内存中被锁定，并且不会被换出。
> 预分配的连续页面不能用于除共享内存(如：SGA)之外的任何其他用途。
> 由于更大的页面大小，降低了内核维护虚拟内存的工作量。

### 在 Linux 上配置大页内存

系统中需要按照下面的步骤完成大页内存的配置

1.  运行下面的命令确定内存是否支持大页内存

    ```console
    $ grep Huge /proc/meminfo
    ```

2.  一些 Linux 系统默认不支持大页内存。对于这些系统，使用 CONFIG_HUGETLBFS 和 CONFIG_HUGETLB_PAGE 配置选项来 build Linux 内核。CONFIG_HUGETLBFS 位于文件系统之下，而当 CONFIG_HUGETLBFS 被选择后，CONFIG_HUGETLB_PAGE 会被选中。

3.  编辑/etc/security/limits.conf 文件中的 memlock，memlock 设置单位为 KB，开启大页内存的情况下，最大的锁定内存限制应至少为当前内存的 90%，关闭大页内存的情况下，应至少为 3145728KB(3GB)。例如，系统有 64GB 内存，则应添加下面的条目来增加最大锁定内存的地址空间。

    - /etc/security/limits.conf

    ```
    * soft memlock 60397977
    * hard memlock 60397977
    ```

    也可以将 memlock 的值设置得比 SGA 需求更大一些。

4.  以 oracle 用户登陆运行 unlimit -l 命令来验证 memlock 是否生效

    ```console
    $ ulimit -l
    ```

5.  运行下面的命令来显示 Hugepagesize 变量值

    ```console
    $ grep Hugepagesize /proc/meminfo
    ```

6.  按照下面的方式创建一个脚本，用来针对当前共享内存段计算 Hugepages 的推荐值

    a. 创建一个文本文件命名为 hugepages_settings.sh，粘贴如下脚本

    ```shell
    #!/bin/bash
    #
    # hugepages_settings.sh
    #
    # Linux bash script to compute values for the
    # recommended HugePages/HugeTLB configuration
    # on Oracle Linux
    #
    # Note: This script does calculation for all shared memory
    # segments available when the script is run, no matter it
    # is an Oracle RDBMS shared memory segment or not.
    #
    # This script is provided by Doc ID 401749.1 from My Oracle Support
    # http://support.oracle.com

    # Welcome text

    echo "
    This script is provided by Doc ID 401749.1 from My Oracle Support
    (http://support.oracle.com) where it is intended to compute values for
    the recommended HugePages/HugeTLB configuration for the current shared
    memory segments on Oracle Linux. Before proceeding with the execution please note following:
    * For ASM instance, it needs to configure ASMM instead of AMM.
    * The 'pga_aggregate_target' is outside the SGA and
      you should accommodate this while calculating the overall size.
    * In case you changes the DB SGA size,
      as the new SGA will not fit in the previous HugePages configuration,
      it had better disable the whole HugePages,
      start the DB with new SGA size and run the script again.
    And make sure that:
    * Oracle Database instance(s) are up and running
    * Oracle Database 11g Automatic Memory Management (AMM) is not setup
      (See Doc ID 749851.1)
    * The shared memory segments can be listed by command:
        # ipcs -m

    Press Enter to proceed..."

    read
    # Check for the kernel version
    KERN=`uname -r | awk -F. '{ printf("%d.%d\n",$1,$2); }'`

    # Find out the HugePage size
    HPG_SZ=`grep Hugepagesize /proc/meminfo | awk '{print $2}'`

    if [ -z "$HPG_SZ" ];then
        echo "The hugepages may not be supported in the system where the script is being executed."
        exit 1
    fi

    # Initialize the counter
    NUM_PG=0

    # Cumulative number of pages required to handle the running shared memory segments
    for SEG_BYTES in `ipcs -m | cut -c44-300 | awk '{print $1}' | grep "[0-9][0-9]*"`
    do
        MIN_PG=`echo "$SEG_BYTES/($HPG_SZ*1024)" | bc -q`
        if [ $MIN_PG -gt 0 ]; then
            NUM_PG=`echo "$NUM_PG+$MIN_PG+1" | bc -q`
        fi
    done

    RES_BYTES=`echo "$NUM_PG * $HPG_SZ * 1024" | bc -q`

    # An SGA less than 100MB does not make sense
    # Bail out if that is the case

    if [ $RES_BYTES -lt 100000000 ]; then
        echo "***********"
        echo "** ERROR **"
        echo "***********"
        echo "Sorry! There are not enough total of shared memory segments allocated for
    HugePages configuration. HugePages can only be used for shared memory segments
    that you can list by command:
        # ipcs -m

    of a size that can match an Oracle Database SGA. Please make sure that:
    * Oracle Database instance is up and running
    * Oracle Database 11g Automatic Memory Management (AMM) is not configured"
        exit 1
    fi

    # Finish with results
    case $KERN in
        '2.4') HUGETLB_POOL=`echo "$NUM_PG*$HPG_SZ/1024" | bc -q`;
              echo "Recommended setting: vm.hugetlb_pool = $HUGETLB_POOL" ;;
        '2.6') echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
        '3.8') echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
        '3.10') echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
        '4.1') echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
        '4.14') echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
        *) echo "Kernel version $KERN is not supported by this script (yet). Exiting." ;;
    esac

    # End
    ```

    b. 运行下面的命令改变文件的权限

    ```console
    $ chmod +x hugepages_settings.sh
    ```

7.  运行 hugepages_settings.sh 脚本来计算出 hugepages 推荐值，运行脚本之前，确保所有使用 hugepages 的应用程序都运行

    ```console
    $ ./hugepages_settings.sh
    ```

8.  根据刚刚计算出来的推荐值设置下面的内核参数

    ```console
    # sysctl -w vm.nr_hugepages=value
    ```

9.  为了确保系统重启后大页内存被分配，增加下面的条目到/etc/sysctl.conf 文件，value 位置填写前面计算得到的推荐值

    - /etc/sysctl.conf

    ```console
    vm.nr_hugepages=value
    ```

10. 运行下面的命令来检查 hugepages 变量的值

    ```console
    $ grep Huge /proc/meminfo
    ```

11. 重启实例

12. 运行下面的命令来检查 hugepages 变量的值。如果你不能通过 nr_hugepages 进行大页内存分配，那么你的可用内存可能存在碎片，可以尝试重启服务器来解决。

    ```console
    $ grep Huge /proc/meminfo
    ```

### 配置大页内存的一些限制条件

大页内存有如下限制：

你必须取消 MEMORY_TARGET 和 MEMORY_MAX_TARGET 初始化参数设置。例如，需要取消实例的初始化参数，可以使用命令 alter system reset

AMM 和大页内存互不兼容，当使用 AMM，整个 SGA 内存空间被分配到/dev/shm 下。当 Oracle 数据库通过 AMM 分配 SGA，大页内存不会保留。为了在 Oracle 数据库上使用大页内存，你必须禁用 AMM。

如果在 32 位环境下使用了 VLM，则你不能使用大页内存作为数据库的 buffer cache。你可以使用大页内存作为其它 SGA 组件(例如 shared_pool,large_pool)，利用共享内存文件系统(ramfs/tmpfs/shmfs)，VLM(buffer cache)内存空间已经被分配。内存文件系统不会保留或使用大页内存。

大页内存不受限于系统启动后的分配和释放，除非系统管理员改变了大页内存的配置，或者是修改可用页面的数量，或者是修改 pool 的大小。如果在系统启动阶段需要的空间没有被保留，则大页内存分配失败。

确保大页内存被合理的配置，如果过多的应用程序没有使用大页内存，系统可能会耗尽内存。

如果实例启动时没有足够的大页内存，并且初始化参数 use_large_pages 设置为 only，则数据库启动失败，并且 alert 告警日志中会提供关于大页内存的必要信息。

### 禁用透明大页

Oracle 推荐在安装之前禁用透明大页。

透明大页不同于标准大页内存因为内核线程 khugepaged 是在运行期间动态分配。而标准大页内存是在启动期间预分配，运行期间不会改变。

尽管透明大页从 UEK2 和 UEK 之后的内核版本默认禁用，但在你的 Linux 系统中有可能默认是启用的。

透明大页在 RHEL6、RHEL7、SUSE11、Oracle Linux6、Oracle Linux7 早先版本(均属于 UEK2 内核)默认是启用的。

透明大页会导致在运行期间内存分配延迟。为了避免性能问题，Oracle 推荐在所有的 Oracle 数据库服务器上禁用透明大页。Oracle 推荐使用标准大页内存来提高性能。

利用 root 用户通过下面的命令来检查透明大页是否启用

- RHEL 内核

  ```console
  # cat /sys/kernel/mm/redhat_transparent_hugepage/enabled
  ```

- 其它内核

  ```console
  # cat /sys/kernel/mm/transparent_hugepage/enabled
  ```

- 下面输出的例子表明透明大页被启用

  ```console
  [always] never
  ```

如果透明大页在系统中被移除，则不会存在/sys/kernel/mm/transparent_hugepage 或/sys/kernel/mm/redhat_transparent_hugepage 文件。

禁用透明大页的方法：

- Oracle Linux6 或早期的版本，添加下面的内容到/etc/grub.conf 文件

  - /etc/grub.conf

  ```
  transparent_hugepage=never
  ```

- 例如

  - /etc/grub.conf

  ```
  title Oracle Linux Server (2.6.32-300.25.1.el6uek.x86_64)
  ```

文件名可能与 Oracle Linux7 或其它系统不同。检查各自的操作系统文档获得准确的文件名，按步骤关闭透明大页。

例如，在 Oracle Linux7.3 系统中，关闭透明大页需要编辑/etc/default/grub 文件，然后运行 grub2-mkconfig 命令。

最后，重启操作系统使配置永久生效。
