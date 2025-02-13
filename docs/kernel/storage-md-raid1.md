# raid1

## 问题
1. `_wait_barrier` : r1conf::nr_pending 被反复增加减少好多次，这是在干嘛呀

## raid1 使用尝试

sudo mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

```txt
➜  ~ mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Thu Apr 13 14:53:57 2023
        Raid Level : raid1
     Used Dev Size : 1022976 (999.00 MiB 1047.53 MB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Thu Apr 13 14:53:57 2023
             State : active, FAILED, Not Started
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : unknown

              Name : localhost.localdomain:0  (local to host localhost.localdomain)
              UUID : 9455c367:d88c7a01:3a034e4e:011dfe96
            Events : 0

    Number   Major   Minor   RaidDevice State
       -       0        0        0      removed
       -       0        0        1      removed

       -       8       32        1      sync   /dev/sdc
       -       8        0        0      sync   /dev/sda
```

- [ ] 似乎是可以设置 Consistency Policy 的

似乎建立的 md0 是存在问题的:
```txt
➜  ~ mkfs.ext4 /dev/md0
mke2fs 1.46.4 (18-Aug-2021)
mkfs.ext4: Device size reported to be zero.  Invalid partition specified, or
        partition table wasn't reread after running fdisk, due to
        a modified partition being busy and in use.  You may need to reboot
        to re-read your partition table.
```

https://unix.stackexchange.com/questions/332061/remove-drive-from-soft-raid

## 删除一个设备
mdadm /dev/md0 --fail /dev/sdc1
mdadm /dev/md0 --remove /dev/sdc1

## 解散
mdadm -S /dev/md0

## 创建
mdadm --create /dev/md0 --name=martins3 --level=mirror --force --raid-devices="/dev/sda1 /dev/sdb1"
mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda1 /dev/sdc1

- [ ] 一个 partition 已经组成了 raid 想不到还是可以直接写，这样是处于什么考虑

## 增加一个设备
mdadm --add /dev/md0 /dev/sdd1
或者
mdadm --manage /dev/md0 --add /dev/vda
mdadm --grow --raid-devices=3 /dev/md0
mdadm --wait /dev/md0

```txt
➜  ~ cat /proc/mdstat
Personalities : [raid1]
md0 : active raid1 sdd1[3](S) sdc1[2] sda1[0]
      1020928 blocks super 1.2 [2/2] [UU]

unused devices: <none>
```

### 为什么增加设备是两个动作呀

```txt
/dev/md10:
           Version : 1.2
     Creation Time : Thu Apr 20 16:48:53 2023
        Raid Level : raid1
        Array Size : 974848 (952.00 MiB 998.24 MB)
     Used Dev Size : 974848 (952.00 MiB 998.24 MB)
      Raid Devices : 1
     Total Devices : 1
       Persistence : Superblock is persistent

       Update Time : Thu Apr 20 16:49:13 2023
             State : clean
    Active Devices : 1
   Working Devices : 1
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : resync

              Name : localhost.localdomain:10  (local to host localhost.localdomain)
              UUID : 4ee51c30:88e90ae8:1084fc8a:96826272
            Events : 23

    Number   Major   Minor   RaidDevice State
       1       8       33        0      active sync   /dev/sdc1


```

```txt
dev/md10:
           Version : 1.2
     Creation Time : Thu Apr 20 16:48:53 2023
        Raid Level : raid1
        Array Size : 974848 (952.00 MiB 998.24 MB)
     Used Dev Size : 974848 (952.00 MiB 998.24 MB)
      Raid Devices : 1
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Thu Apr 20 16:49:23 2023
             State : clean
    Active Devices : 1
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 1

Consistency Policy : resync

              Name : localhost.localdomain:10  (local to host localhost.localdomain)
              UUID : 4ee51c30:88e90ae8:1084fc8a:96826272
            Events : 24

    Number   Major   Minor   RaidDevice State
       1       8       33        0      active sync   /dev/sdc1

       2       8        1        -      spare   /dev/sda1
```

```txt
/dev/md10:
           Version : 1.2
     Creation Time : Thu Apr 20 16:48:53 2023
        Raid Level : raid1
        Array Size : 974848 (952.00 MiB 998.24 MB)
     Used Dev Size : 974848 (952.00 MiB 998.24 MB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Thu Apr 20 16:49:23 2023
             State : active, degraded
    Active Devices : 1
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 1

Consistency Policy : resync

              Name : localhost.localdomain:10  (local to host localhost.localdomain)
              UUID : 4ee51c30:88e90ae8:1084fc8a:96826272
            Events : 26

    Number   Major   Minor   RaidDevice State
       1       8       33        0      active sync   /dev/sdc1
       -       0        0        1      removed

       2       8        1        -      spare   /dev/sda1
```
被管理的，未必是一定被使用的，但是为什么需要这样被设计呀？

### 第一步，增加 disk

## reshape 的过程中，是如何知道是增加什么 disk 呀?


## 一些 backtrace

### 删除

mdadm /dev/md0 --fail /dev/sdc1

这一个命令会船舰这两个:
```txt
#0  raid1_error (mddev=0xffff888107dcc000, rdev=0xffff888105c3d400) at drivers/md/raid1.c:1660
#1  0xffffffff81e1cee9 in md_error (mddev=mddev@entry=0xffff888107dcc000, rdev=0xffff888105c3d400) at drivers/md/md.c:7975
#2  0xffffffff81e26889 in md_error (rdev=<optimized out>, mddev=0xffff888107dcc000) at drivers/md/md.c:7970
#3  set_disk_faulty (dev=8388609, mddev=0xffff888107dcc000) at drivers/md/md.c:7427
#4  md_ioctl (bdev=0xffff888100490600, mode=<optimized out>, cmd=2345, arg=<optimized out>) at drivers/md/md.c:7569
#5  0xffffffff81754ed7 in blkdev_ioctl (file=<optimized out>, cmd=2345, arg=2049) at block/ioctl.c:615
#6  0xffffffff8143d13b in vfs_ioctl (arg=2049, cmd=<optimized out>, filp=0xffff888106c4f000) at fs/ioctl.c:51
#7  __do_sys_ioctl (arg=2049, cmd=<optimized out>, fd=<optimized out>) at fs/ioctl.c:870
#8  __se_sys_ioctl (arg=2049, cmd=<optimized out>, fd=<optimized out>) at fs/ioctl.c:856
#9  __x64_sys_ioctl (regs=<optimized out>) at fs/ioctl.c:856
#10 0xffffffff82286f9c in do_syscall_x64 (nr=<optimized out>, regs=0xffffc900015abf58) at arch/x86/entry/common.c:50
#11 do_syscall_64 (regs=0xffffc900015abf58, nr=<optimized out>) at arch/x86/entry/common.c:80
```

```txt
#0  raid1_remove_disk (mddev=0xffff888107dcc000, rdev=0xffff888105c3d400) at drivers/md/raid1.c:1844
#1  0xffffffff81e1d8c7 in remove_and_add_spares (mddev=mddev@entry=0xffff888107dcc000, this=this@entry=0x0 <fixed_percpu_data>) at drivers/md/md.c:9163
#2  0xffffffff81e25839 in md_check_recovery (mddev=mddev@entry=0xffff888107dcc000) at drivers/md/md.c:9408
#3  0xffffffff81e11dde in raid1d (thread=<optimized out>) at drivers/md/raid1.c:2564
#4  0xffffffff81e1a062 in md_thread (arg=0xffff888105d98c60) at drivers/md/md.c:7903
#5  0xffffffff8116ea09 in kthread (_create=0xffff888105b40e00) at kernel/kthread.c:376
#6  0xffffffff810029b9 in ret_from_fork () at arch/x86/entry/entry_64.S:308
```

mdadm /dev/md0 --remove /dev/sdc1

## 文档的记录

1. 活动 : resync, recovery, or reshape activity
  - resync ?

```txt
       --backup-file=
              This  is  needed  when  --grow  is  used  to increase the number of raid-devices in a RAID5
              or RAID6 if there are no spare devices available, or to shrink, change RAID level or layout.
              See the GROW MODE section below on RAID-DEVICES CHANGES.  The file must be stored on a separate device, not on the RAID array being reshaped.
```

## raid1_personality

```c
static struct md_personality raid1_personality =
{
	.name		= "raid1",
	.level		= 1,
	.owner		= THIS_MODULE,
	.make_request	= raid1_make_request,
	.run		= raid1_run,
	.free		= raid1_free,
	.status		= raid1_status,
	.error_handler	= raid1_error,
	.hot_add_disk	= raid1_add_disk,
	.hot_remove_disk= raid1_remove_disk,
	.spare_active	= raid1_spare_active,
	.sync_request	= raid1_sync_request,
	.resize		= raid1_resize,
	.size		= raid1_size,
	.check_reshape	= raid1_reshape,
	.quiesce	= raid1_quiesce,
	.takeover	= raid1_takeover,
};
```
好像也没什么可以看的。

`raid1_make_request` 中注册了 `raid1_end_write_request`

## r1conf

```c
struct r1conf {
	struct mddev		*mddev;
	struct raid1_info	*mirrors;	/* twice 'raid_disks' to
						 * allow for replacements.
						 */
	int			raid_disks;

	spinlock_t		device_lock;

	/* list of 'struct r1bio' that need to be processed by raid1d,
	 * whether to retry a read, writeout a resync or recovery
	 * block, or anything else.
	 */
	struct list_head	retry_list;
	/* A separate list of r1bio which just need raid_end_bio_io called.
	 * This mustn't happen for writes which had any errors if the superblock
	 * needs to be written.
	 */
	struct list_head	bio_end_io_list;

	/* queue pending writes to be submitted on unplug */
	struct bio_list		pending_bio_list;

	/* for use when syncing mirrors:
	 * We don't allow both normal IO and resync/recovery IO at
	 * the same time - resync/recovery can only happen when there
	 * is no other IO.  So when either is active, the other has to wait.
	 * See more details description in raid1.c near raise_barrier().
	 */
	wait_queue_head_t	wait_barrier;
	spinlock_t		resync_lock;
	atomic_t		nr_sync_pending;
	atomic_t		*nr_pending;
	atomic_t		*nr_waiting;
	atomic_t		*nr_queued;
	atomic_t		*barrier;
	int			array_frozen;

	/* Set to 1 if a full sync is needed, (fresh device added).
	 * Cleared when a sync completes.
	 */
	int			fullsync;

	/* When the same as mddev->recovery_disabled we don't allow
	 * recovery to be attempted as we expect a read error.
	 */
	int			recovery_disabled;

	/* poolinfo contains information about the content of the
	 * mempools - it changes when the array grows or shrinks
	 */
	struct pool_info	*poolinfo;
	mempool_t		r1bio_pool;
	mempool_t		r1buf_pool;

	struct bio_set		bio_split;

	/* temporary buffer to synchronous IO when attempting to repair
	 * a read error.
	 */
	struct page		*tmppage;

	/* When taking over an array from a different personality, we store
	 * the new thread here until we fully activate the array.
	 */
	struct md_thread	*thread;

	/* Keep track of cluster resync window to send to other
	 * nodes.
	 */
	sector_t		cluster_sync_low;
	sector_t		cluster_sync_high;

};
```

## 如何理解其中的 barrier
- raid1_reshape
  - freeze_array(conf, 1 ) :
  - 更新 conf->raid_disks = mddev->raid_disks = raid_disks;
  - unfreeze_array(conf);

- wait_read_barrier
- wait_barrier

- 机制加入的时间 : fd76863e37fef26fe05547fddfa6e3d05e1682e6

思考一个很牛逼的场景

## 第一个 freeze_array 还没执行完成，结果第二个开始执行了？
- [ ] freeze_array 到底是阻碍别人，还是别别人阻碍？

## 提交一个 bio，还没返回的时候，首先将一个 devices 删除，然后新增一个新的 devices



## 基本的数据结构
```c
	struct r1conf *conf = mddev->private;

	conf->mirrors = newmirrors;
	kfree(conf->poolinfo);
	conf->poolinfo = newpoolinfo;
```
mddev 是 generic 的，而 r1conf 并不是


## io 返回的场景
```txt
18660 [  998.657409]  [<ffffffffc042abee>] free_r1bio+0x5e/0x80 [raid1]
18661 [  998.657826]  [<ffffffffc042ad68>] close_write+0xb8/0xc0 [raid1]
18662 [  998.658228]  [<ffffffffc042b3e5>] r1_bio_write_done+0x25/0x50 [raid1]
18663 [  998.658664]  [<ffffffffc042bc78>] raid1_end_write_request+0x118/0x2f0 [raid1] # 注册的 hook
18664 [  998.659150]  [<ffffffffc0137b57>] ? ata_scsi_qc_complete+0x67/0x450 [libata]
18665 [  998.659622]  [<ffffffff8128d83c>] bio_endio+0x8c/0x130
18666 [  998.659969]  [<ffffffff81355a40>] blk_update_request+0x90/0x370
18667 [  998.660374]  [<ffffffff814ed944>] scsi_end_request+0x34/0x1e0
18668 [  998.660763]  [<ffffffff814edcb8>] scsi_io_completion+0x168/0x720
18669 [  998.661174]  [<ffffffffc0145053>] ? __ata_sff_port_intr+0xa3/0x130 [libata]
18670 [  998.661757]  [<ffffffff814e2fac>] scsi_finish_command+0xdc/0x140
18671 [  998.662164]  [<ffffffff814ed200>] scsi_softirq_done+0x130/0x160
18672 [  998.662569]  [<ffffffff8135d3c6>] blk_done_softirq+0x96/0xc0
18673 [  998.662955]  [<ffffffff810a4c15>] __do_softirq+0xf5/0x280
18674 [  998.663326]  [<ffffffff817984ec>] call_softirq+0x1c/0x30
18675 [  998.663690]  [<ffffffff8102f715>] do_softirq+0x65/0xa0
18676 [  998.664041]  [<ffffffff810a4f95>] irq_exit+0x105/0x110
18677 [  998.664384]  [<ffffffff81799936>] do_IRQ+0x56/0xf0
18678 [  998.664711]  [<ffffffff8178b36a>] common_interrupt+0x16a/0x16a
```

最后的位置:
```c
static void put_all_bios(struct r1conf *conf, struct r1bio *r1_bio)
{
	int i;

	for (i = 0; i < conf->raid_disks * 2; i++) {
		struct bio **bio = r1_bio->bios + i;
		if (!BIO_SPECIAL(*bio))
			bio_put(*bio);
		*bio = NULL;
	}
}

static void free_r1bio(struct r1bio *r1_bio)
{
	struct r1conf *conf = r1_bio->mddev->private;

	put_all_bios(conf, r1_bio);
	mempool_free(r1_bio, &conf->r1bio_pool);
}
```

其实并不会，因为 `raid1_end_write_request`

## 在 raid1 中，每一个 r1bio 在每一个 disk 中都会持有一个对应的 io
- [ ] 需要等到所有人返回的再返回吧?


## [ ] 为什么总是乘以 2 :  disks = conf->raid_disks * 2

- `raid1_reshape`
```c
    // 导致 r1bio 中持有的 2 倍的 bio
	newpoolinfo->raid_disks = raid_disks * 2;

	ret = mempool_init(&newpool, NR_RAID_BIOS, r1bio_pool_alloc,
			   rbio_pool_free, newpoolinfo);

    // 导致 r1conf 中持有的 raid1_info::rdev 是 disk 的两倍
	newmirrors = kzalloc(array3_size(sizeof(struct raid1_info),
					 raid_disks, 2),
			     GFP_KERNEL);
```

- 为什么 put_all_bios 中会出现所有的 bio 都是 NULL


常见模式
```txt
[  424.962244] . .
[  424.962441] . .
[  424.962640] . .
[  424.973235] x .
[  491.093996] . . . .
[  491.094225] . . . .
[  491.094445] . . . .
[  491.094666] . . . .
[  491.094888] . . . .
```
其实后面两个从来没有赋值。

很多 io 其实都是一个设备，甚至没有

- raid1_end_write_request
  - r1_bio_write_done
    - raid_end_bio_io
      - call_bio_endio
      - allow_barrier
      - free_r1bio

不知道出于何种原因，在 `raid1_end_write_request` 中会 `r1_bio->bios[mirror] = NULL`，
表示直接 bio 结束了。

不太理解，难道不需要释放吗？

## r1bio 的生命周期

`r1bio_pool_alloc` : 组件初始化

```c
	newpoolinfo->mddev = mddev;
	newpoolinfo->raid_disks = raid_disks * 2;

	ret = mempool_init(&newpool, NR_RAID_BIOS, r1bio_pool_alloc,
			   rbio_pool_free, newpoolinfo);
```
r1bio 构成的数据也是两倍的。

- `raid1_write_request` : 创建

## raid1 的状态

- `1BIO_Returned`

## [ ] 当有的盘拔掉后，那些 bio 是在哪里被清理的

```c
/*
 * this is our 'private' RAID1 bio.
 *
 * it contains information about what kind of IO operations were started
 * for this RAID1 operation, and about their status:
 */

struct r1bio {
	atomic_t		remaining; /* 'have we finished' count,
					    * used from IRQ handlers
					    */
	atomic_t		behind_remaining; /* number of write-behind ios remaining
						 * in this BehindIO request
						 */
    }
```

- `raid1_end_write_request`
  - find_bio_disk : 必须找到对应的 mirror，但是
  - r1_bio_write_done : 根据 `r1_bio->remaining` 来分析


- raid1_write_request
  - atomic_inc(&r1_bio->remaining); :

## 如何理解 raid1_remove_disk

哇，完全无法理解这个逻辑，我靠

```c
    // 所有的逻辑都在这个前提下:
	if (rdev == p->rdev) {
```


- raid1_remove_disk
  - freeze_array
    - [ ] 这个是等待所有的 io 都返回吗？

freeze_array 只是将 io 放到 queue 中间:
```c
static void freeze_array(struct r1conf *conf, int extra)
{
	/* Stop sync I/O and normal I/O and wait for everything to
	 * go quiet.
	 * This is called in two situations:
	 * 1) management command handlers (reshape, remove disk, quiesce).
	 * 2) one normal I/O request failed.

	 * After array_frozen is set to 1, new sync IO will be blocked at
	 * raise_barrier(), and new normal I/O will blocked at _wait_barrier()
	 * or wait_read_barrier(). The flying I/Os will either complete or be
	 * queued. When everything goes quite, there are only queued I/Os left.

	 * Every flying I/O contributes to a conf->nr_pending[idx], idx is the
	 * barrier bucket index which this I/O request hits. When all sync and
	 * normal I/O are queued, sum of all conf->nr_pending[] will match sum
	 * of all conf->nr_queued[]. But normal I/O failure is an exception,
	 * in handle_read_error(), we may call freeze_array() before trying to
	 * fix the read error. In this case, the error read I/O is not queued,
	 * so get_unqueued_pending() == 1.
	 *
	 * Therefore before this function returns, we need to wait until
	 * get_unqueued_pendings(conf) gets equal to extra. For
	 * normal I/O context, extra is 1, in rested situations extra is 0.
	 */
	spin_lock_irq(&conf->resync_lock);
	conf->array_frozen = 1;
	raid1_log(conf->mddev, "wait freeze");
	wait_event_lock_irq_cmd(
		conf->wait_barrier,
		get_unqueued_pending(conf) == extra,
		conf->resync_lock,
		flush_pending_writes(conf));
	spin_unlock_irq(&conf->resync_lock);
}
```
直接提交为 `submit_bio_noacct` 而已。


在 reshape 中也是使用 `freeze_array` 和 `unfreeze_array` 来保护。

深入理解一下 `_wait_barrier` 吧！

## barrier
同步应该不是整个盘停下来的，而是 sector 级别的，如果正在对于一个 sector 同步，那么就不要让另一个 thread 写

```c
	/* for use when syncing mirrors:
	 * We don't allow both normal IO and resync/recovery IO at
	 * the same time - resync/recovery can only happen when there
	 * is no other IO.  So when either is active, the other has to wait.
	 * See more details description in raid1.c near raise_barrier().
	 */
	wait_queue_head_t	wait_barrier;
	spinlock_t		resync_lock;
	atomic_t		nr_sync_pending;
	atomic_t		*nr_pending;      // 表示是当时的 normal io 的数量
	atomic_t		*nr_waiting;
	atomic_t		*nr_queued;       // 失败的 io , 放到 raid1d 来重新尝试的列表
	atomic_t		*barrier;         // raise_barrier 来增加 1
	int			array_frozen;         // 当前 array 是否在 frozen
```

```c
/* Barriers....
 * Sometimes we need to suspend IO while we do something else,
 * either some resync/recovery, or reconfigure the array.
 * To do this we raise a 'barrier'.
 * The 'barrier' is a counter that can be raised multiple times
 * to count how many activities are happening which preclude
 * normal IO.
 * We can only raise the barrier if there is no pending IO.
 * i.e. if nr_pending == 0.
 * We choose only to raise the barrier if no-one is waiting for the
 * barrier to go down.  This means that as soon as an IO request
 * is ready, no other operations which require a barrier will start
 * until the IO request has had a chance.
 *
 * So: regular IO calls 'wait_barrier'.  When that returns there
 *    is no backgroup IO happening,  It must arrange to call
 *    allow_barrier when it has finished its IO.
 * backgroup IO calls must call raise_barrier.  Once that returns
 *    there is no normal IO happeing.  It must arrange to call
 *    lower_barrier when the particular background IO completes.
 *
 * If resync/recovery is interrupted, returns -EINTR;
 * Otherwise, returns 0.
 */
static int raise_barrier(struct r1conf *conf, sector_t sector_nr)
```

- `RESYNC_DEPTH`

### `_wait_barrier` 中的同步技术

- [ ] 至少两个 read 在一起，确实非常奇怪

```c
	if (!READ_ONCE(conf->array_frozen) &&
	    !atomic_read(&conf->barrier[idx]))
		return ret;
```
这会让 `freeze_array` 和 resync 无法并发执行。

### nr_sync_pending

- raid1_sync_request
  - raise_barrier
    - atomic_inc(&conf->nr_sync_pending);

- put_sync_write_buf
  - put_buf
    - lower_barrier

### 什么时候会 `freeze_array`

当 `freeze_array` 是需要阻碍住所有的文件:

- raid1_remove_disk
- handle_write_finished 被 raid1d 调用
- raid1_reshape
- raid1_quiesce

是不是存在 io 的时候，就没有办法 sync 内存吗?

使用 `get_unqueued_pending` 来判断的: `nr_pending` - `nr_queued` 等于 0 的时候。


r1conf:nr_pending 增加的位置：
- 在 `raid1_write_request` -> `wait_barrier` -> `_wait_barrier` 中，增加 r1conf::nr_pending
- wait_read_barrier


r1conf:nr_pending 减少的位置：
- `raid_end_bio_io` -> `allow_barrier`

所以 `nr_pending` 的含义就是还写写入到盘中的数据。


r1conf::nr_queued 增加的位置:
- reschedule_retry : 其调用位置
  * raid1_end_read_request
  * r1_bio_write_done
  * put_sync_write_buf
  * handle_write_finished : 似乎没什么人调用

在 raid1d 中 conf::nr_queued 会减少，所以 queue 的含义很清楚了，就是 r1conf::retry_list 中的大小

所以，freeze_array 的时候，之前的 bio 都返回了。

### [x] 为什么要使用一个 page 来同步
因为只是 page 的

### resync/recovery 行为是什么?

## raid1d 是做什么的?
- 每创建一个 raid 的时候，需要增加一个

- md_thread

reschedule_retry 中将无法发送下去的 r1bio 放到 r1conf::retry_list 中
如上所说，`reschedule_retry` 的调用位置就是 `end_bio` 的位置了:

- raid1d
  - r1conf::retry_list 中取出一个 r1bio 出来
  - atomic_dec(&conf->nr_queued[idx]);
  - 针对情况分别提交下去:
    - handle_sync_write_finished
    - sync_request_write
    - handle_write_finished
    - handle_read_error

`reschedule_retry` 中的代码为什么不会导致 race condition 啊!

```txt
#0  reschedule_retry (r1_bio=0xffff888107995d00) at drivers/md/raid1.c:295
#1  0xffffffff8174bf01 in req_bio_endio (error=0 '\000', nbytes=65536, bio=0xffff8881145ad800, rq=0xffff88810b5e7840) at block/blk-mq.c:795
#2  blk_update_request (req=req@entry=0xffff88810b5e7840, error=error@entry=0 '\000', nr_bytes=458752, nr_bytes@entry=720896) at block/blk-mq.c:927
#3  0xffffffff81ba20a7 in scsi_end_request (req=req@entry=0xffff88810b5e7840, error=error@entry=0 '\000', bytes=bytes@entry=720896) at drivers/scsi/scsi_lib.c:538
#4  0xffffffff81ba2bee in scsi_io_completion (cmd=0xffff88810b5e7948, good_bytes=720896) at drivers/scsi/scsi_lib.c:976
#5  0xffffffff81bc8be0 in virtscsi_vq_done (fn=0xffffffff81bc89a0 <virtscsi_complete_cmd>, virtscsi_vq=0xffff88810a675b68, vscsi=0xffff88810a675810) at drivers/scsi/virtio_scsi.c:183
#6  virtscsi_req_done (vq=<optimized out>) at drivers/scsi/virtio_scsi.c:198
#7  0xffffffff818b39ab in vring_interrupt (irq=<optimized out>, _vq=0xffff888107995d00) at drivers/virtio/virtio_ring.c:2491
#8  vring_interrupt (irq=<optimized out>, _vq=0xffff888107995d00) at drivers/virtio/virtio_ring.c:2466
#9  0xffffffff811c926d in __handle_irq_event_percpu (desc=desc@entry=0xffff88810ae01e00) at kernel/irq/handle.c:158
#10 0xffffffff811c9478 in handle_irq_event_percpu (desc=0xffff88810ae01e00) at kernel/irq/handle.c:193
#11 handle_irq_event (desc=desc@entry=0xffff88810ae01e00) at kernel/irq/handle.c:210
#12 0xffffffff811ce39b in handle_edge_irq (desc=0xffff88810ae01e00) at kernel/irq/chip.c:819
#13 0xffffffff810d9d9f in generic_handle_irq_desc (desc=0xffff88810ae01e00) at ./include/linux/irqdesc.h:158
#14 handle_irq (regs=<optimized out>, desc=0xffff88810ae01e00) at arch/x86/kernel/irq.c:231
#15 __common_interrupt (regs=<optimized out>, vector=341497856) at arch/x86/kernel/irq.c:250
#16 0xffffffff82290e7f in common_interrupt (regs=0xffffc9000014fe38, error_code=<optimized out>) at arch/x86/kernel/irq.c:240
```
来源是 `end_sync_read`

- req_bio_endio
  - bio_endio
    - end_sync_read : 创建的 md 阵列的时候会使用这部分代码

## recovery_flags 是什么状态？

```c
enum recovery_flags {
	/*
	 * If neither SYNC or RESHAPE are set, then it is a recovery.
	 */
	MD_RECOVERY_RUNNING,	/* a thread is running, or about to be started */
	MD_RECOVERY_SYNC,	/* actually doing a resync, not a recovery */
	MD_RECOVERY_RECOVER,	/* doing recovery, or need to try it. */
	MD_RECOVERY_INTR,	/* resync needs to be aborted for some reason */
	MD_RECOVERY_DONE,	/* thread is done and is waiting to be reaped */
	MD_RECOVERY_NEEDED,	/* we might need to start a resync/recover */
	MD_RECOVERY_REQUESTED,	/* user-space has requested a sync (used with SYNC) */
	MD_RECOVERY_CHECK,	/* user-space request for check-only, no repair */
	MD_RECOVERY_RESHAPE,	/* A reshape is happening */
	MD_RECOVERY_FROZEN,	/* User request to abort, and not restart, any action */
	MD_RECOVERY_ERROR,	/* sync-action interrupted because io-error */
	MD_RECOVERY_WAIT,	/* waiting for pers->start() to finish */
	MD_RESYNCING_REMOTE,	/* remote node is running resync thread */
};
```

## end_sync_write : 难道同步写和异步的差别需要 raid1 来处理？

这个应该不是 aio 的 sync 吧，而是普通的 synchronous write 的场景。

```txt
#0  end_sync_write (bio=0xffff8881093ea200) at drivers/md/raid1.c:1962
#1  0xffffffff8174bb11 in req_bio_endio (error=0 '\000', nbytes=65536, bio=0xffff8881093ea200, rq=0xffff88810b472d80) at block/blk-mq.c:795
#2  blk_update_request (req=req@entry=0xffff88810b472d80, error=error@entry=0 '\000', nr_bytes=196608, nr_bytes@entry=262144) at block/blk-mq.c:927
#3  0xffffffff81ba1d47 in scsi_end_request (req=req@entry=0xffff88810b472d80, error=error@entry=0 '\000', bytes=bytes@entry=262144) at drivers/scsi/scsi_lib.c:538
#4  0xffffffff81ba288e in scsi_io_completion (cmd=0xffff88810b472e88, good_bytes=262144) at drivers/scsi/scsi_lib.c:976
#5  0xffffffff81bc8880 in virtscsi_vq_done (fn=0xffffffff81bc8640 <virtscsi_complete_cmd>, virtscsi_vq=0xffff88810a750b38, vscsi =0xffff88810a750810) at drivers/scsi/virtio_scsi.c:183
#6  virtscsi_req_done (vq=<optimized out>) at drivers/scsi/virtio_scsi.c:198
#7  0xffffffff818b364b in vring_interrupt (irq=<optimized out>, _vq=0xffff8881093ea200) at drivers/virtio/virtio_ring.c:2491
#8  vring_interrupt (irq=<optimized out>, _vq=0xffff8881093ea200) at drivers/virtio/virtio_ring.c:2466
#9  0xffffffff811c912d in __handle_irq_event_percpu (desc=desc@entry=0xffff88810a562a00) at kernel/irq/handle.c:158
#10 0xffffffff811c9338 in handle_irq_event_percpu (desc=0xffff88810a562a00) at kernel/irq/handle.c:193
#11 handle_irq_event (desc=desc@entry=0xffff88810a562a00) at kernel/irq/handle.c:210
#12 0xffffffff811ce25b in handle_edge_irq (desc=0xffff88810a562a00) at kernel/irq/chip.c:819
#13 0xffffffff810d9d9f in generic_handle_irq_desc (desc=0xffff88810a562a00) at ./include/linux/irqdesc.h:158
#14 handle_irq (regs=<optimized out>, desc=0xffff88810a562a00) at arch/x86/kernel/irq.c:231
#15 __common_interrupt (regs=<optimized out>, vector=155099648) at arch/x86/kernel/irq.c:250
#16 0xffffffff82290dcf in common_interrupt (regs=0xffffc90000137e38, error_code=<optimized out>) at arch/x86/kernel/irq.c:240
Backtrace stopped: Cannot access memory at address 0xffffc90000561010
```

### mbio->bi_end_io	= raid1_end_write_request
```txt
#0  raid1_end_write_request (bio=0xffff888173ad9400) at drivers/md/raid1.c:448
#1  0xffffffff8174bb11 in req_bio_endio (error=0 '\000', nbytes=4096, bio=0xffff888173ad9400, rq=0xffff88810b0b98c0) at block/blk-mq.c:795
#2  blk_update_request (req=req@entry=0xffff88810b0b98c0, error=error@entry=0 '\000', nr_bytes=nr_bytes@entry=4096) at block/blk-mq.c:927
#3  0xffffffff81ba1d47 in scsi_end_request (req=req@entry=0xffff88810b0b98c0, error=error@entry=0 '\000', bytes=bytes@entry=4096) at drivers/scsi/scsi_lib.c:538
#4  0xffffffff81ba288e in scsi_io_completion (cmd=0xffff88810b0b99c8, good_bytes=4096) at drivers/scsi/scsi_lib.c:976
#5  0xffffffff81bc8880 in virtscsi_vq_done (fn=0xffffffff81bc8640 <virtscsi_complete_cmd>, virtscsi_vq=0xffff88810a750a68, vscsi=0xffff88810a750810) at drivers/scsi/virtio_scsi.c:183
#6  virtscsi_req_done (vq=<optimized out>) at drivers/scsi/virtio_scsi.c:198
#7  0xffffffff818b364b in vring_interrupt (irq=<optimized out>, _vq=0xffff888173ad9400) at drivers/virtio/virtio_ring.c:2491
#8  vring_interrupt (irq=<optimized out>, _vq=0xffff888173ad9400) at drivers/virtio/virtio_ring.c:2466
#9  0xffffffff811c912d in __handle_irq_event_percpu (desc=desc@entry=0xffff88810a561000) at kernel/irq/handle.c:158
#10 0xffffffff811c9338 in handle_irq_event_percpu (desc=0xffff88810a561000) at kernel/irq/handle.c:193
#11 handle_irq_event (desc=desc@entry=0xffff88810a561000) at kernel/irq/handle.c:210
#12 0xffffffff811ce25b in handle_edge_irq (desc=0xffff88810a561000) at kernel/irq/chip.c:819
#13 0xffffffff810d9d9f in generic_handle_irq_desc (desc=0xffff88810a561000) at ./include/linux/irqdesc.h:158
#14 handle_irq (regs=<optimized out>, desc=0xffff88810a561000) at arch/x86/kernel/irq.c:231
#15 __common_interrupt (regs=<optimized out>, vector=1940755456) at arch/x86/kernel/irq.c:250
#16 0xffffffff82290dcf in common_interrupt (regs=0xffffc900000cfe38, error_code=<optimized out>) at arch/x86/kernel/irq.c:240
Backtrace stopped: Cannot access memory at address 0xffffc900002a5010
```

## 分析一下 read 的锁

- raid1_read_request : 几乎进去的瞬间上锁
  - `wait_read_barrier`

- raid1_end_read_request : 等到结束的时候
  - raid_end_bio_io
    - allow_barrier : 开锁

write 的时候 : raid1_end_write_request : 类似的

## 有时候存在这个报错

```txt
[ 2862.632964] md: could not open unknown-block(8,1).
[ 2862.633525] md: md_import_device returned -16
[ 2862.634200] md: could not open unknown-block(8,1).
[ 2862.635155] md: md_import_device returned -16
```

## 难道，实际上，考虑到盘的变化?
显然不可能，在 `handle_sync_write_finished` 中

## r1conf::pending_bio_list
raid1_write_request 将 bio 放到该链表中


## 似乎是 bug 的
`handle_write_finished` 中的注释:

```c
		list_add(&r1_bio->retry_list, &conf->bio_end_io_list);
		/*
		 * In case freeze_array() is waiting for condition
		 * get_unqueued_pending() == extra to be true.
		 */
```
如果将 r1bio 挂到列表中，是不能将 `freeze_array` 直接释放的，此事 reshape，
导致所有的 r1bio 全部都释放。

## raid1 容错性测试
运行过程中，一块盘可以掉了，例如是 sda，但是如果此时让 sda 正常，sdb 挂掉，那么 fio 直接出错。

## [ ] mddev::degraded 是做什么的?
给 md 用的

## rcu 是在保护谁的?

`raid1_status`

`print_conf`

似乎是这里的:
```c
rcu_assign_pointer(p->rdev, rdev);
```
`raid1_spare_active`

`raid1_remove_disk`

保护的 rdev ，而不是 mirror ，那么很遗憾

## resync 行为到底是什么行为?

似乎在最开始创建的时候，存在如下的内容:
- raid1d
  - sync_request_write

## md_wait_for_blocked_rdev 这个同步机制是做什么的?
