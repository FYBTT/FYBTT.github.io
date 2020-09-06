---
layout:     post
title:      tracepoint_of_Block_Layer
subtitle:   trace event of block-layer(kernel-4.4.174)
date:       2020-03-19
author:     bear
header-img: img/computer.jpg
catalog:    true
categories: linux
tags:       kernel trace block
--- 
# 基本概念及流程：
bio 代表一个IO 请求。
request 是bio 提交给IO调度器产生的数据，一个request 中放着顺序排列的bio。
当设备提交bio 给IO调度器时，IO调度器可能会插入bio，或者生成新的request。
request_queue代表着一个物理设备，顺序的放着request。
## request的数据结构
![requeststruct][0]
## request的种类
IO从块设备层（block IO layer），到发送到块设备驱动（device driver）整个过程经过三类队列：
1. unplug request queue 属于线程 current->plug
2. elevator queue 调度队列，不同的调度器，队列不同
3. device request queue 派遣队列，dispatch queue。（例如，在deadline_dispatch_requests中实现） 该queue即针对设备的，每个设备都有独有的request queue

plug和unplug：目的是让请求不马上被驱动程序处理。设备处于pluged状态，设备不会被激活。处于unplugged状态，被激活。
来自上层的请求，先尝试合并入unplug 队列，若不能合并，则调用elv_merge合并入调度队列elevator queue（elv_merge的后两种req的获取途径都是从elevator queue中得到）。若找不到可合并的请求，则获得一个空请求request，用该bio初始化该request，然后放到unplug队列中
## request的流动
一个request在送往设备之前会被放入到每个设备所对应的request queue。其实，通过分析一个IO在elevator层其实会经过很多request queue，不同的request queue会有不同的作用。
![requestLevel][1]

# bio/request的提交流程

## task自有的蓄流层
![blk_queue_bio][2]
1. 调用generic_make_request，维护当前的bio链表，调用bdev_get_queue获取设备的request_queue，调用blk_partition_remap检查目标设备是否是一个磁盘分区，如果是个分区，则需要做转化；
2. 调用request_queue的make_request_fn函数，将bio插入到request_queue中；
3. 调用blk_init_allocated_queue阶段绑定的blk_queue_bio来处理第2步传递的bio；
    1. 如果因为硬件限制bio的地址范围和io驱动的数据交互，调用blk_queue_bounce创建bounce buffer来转移bio；
    2. 如果某些原因，例如bio横跨设备边界，调用blk_queue_split将一个bio分割成两个，重新调用generic_make_request处理分割的bio；，调用blk_attempt_plug_merge将bio合并到当前线程的plugged蓄流链表中，遍历request_queue的plug_list链表中的request；
    3. 请求队列允许合并(QUEUE_FLAG_NOMERGES标记)，调用blk_attempt_plug_merge将bio合并到当前线程的plugged蓄流链表中，遍历request_queue的plug_list链表中的request；
        1. 如果发现可以合并的request；
        2. 调用bio_attempt_back_merge或bio_attempt_front_merge进行合并；
![bio_merge][3]
    4. 请求队列不允许合并，申请request_queue的queue_lock，调用调度器的elv_merge函数，进行bio的合并；
![elv_merge][4]
    5. 合并失败的处理逻辑比较简单：调用get_request为bio分配一个新的request结构，注意：这里可能会阻塞(sleep)；
    6. 如果task的蓄流功能启动；
        1. 蓄流链表plug_list为空，启动plug_trace；
        2. 蓄流链表有如果不为空，且超过MAX_REQUEST_COUNT的阈值，调用blk_flush_plug_list，向调度层提交request,启动plug_trace；
        3. 将新请求的request添加到plug_list链表中;
    7. task的蓄流功能未启动，直接提交bio到调度层的request；

## elevator调度层
![blk_flush_plug_list][5]
1. 蓄流层的request数量超过阈值以后开始进行request的提交，blk_flush_plug_list会遍历蓄流链表中的每个request，然后将每个request通过_elv_add_request接口添加到elecator调度队列中，添加的过程中会尝试与调度队列中已有的request进行合并。
    1. 对蓄流链表中的request进行排序
    2. 从蓄流链表中摘下一个request，将request合入到elevator中。对于第一个摘下的request，执行queue_unplug操作。
        1. 调用elv_add_request将request插入到elevator的调度队列中
        2. 添加的过程中会尝试与调度队列中已有的request进行合并，先让蓄流的request调用elv_attempt_insert_merge尝试与调度队列中的request合并，如果不能合并则落入到ELEVATOR_INSERT_SORT分支
2. 当调度器需要发送request时，会调用elevator_dispatch_fn。该函数会直接从调度器所管理的request queue中获取一个request，然后调用elv_dispatch_sort函数将请求加入到设备所在的request queue中(dispatch分发层)，该函数可以对request进行排序，插入到合适的位置。
![unplug][6]

## 设备驱动——dispatch分发层
1. 通过调用elevator的elevator_dispatch_fn函数向dispatch分发层提交request
2. 通过调用 run_queue 的方法将 elevator 分类处理过的 request 转移至 device request queue 中，最后调用 scsi_dispatch_cmd 方法将请求发送到 HBA 。
3. 在这个过程有一些问题需要处理：底层设备可能存在故障； HBA 的处理队列是有长度限制的。因此，如何连续调度 device request queue 及重新调度 request 成了一个需要考虑的问题。在 Linux 中，如果 scsi 层需要重新调度一个 request，可以通过 blk_requeue_request 接口来完成。通过该接口，可以把 request 重新放回到 device request queue 中。另外，在一个 request 结束之后的回调函数中，需要通过 scsi_run_queue 函数来再次调度处理 device request queue 中的剩余请求，从而可以保证批量处理 device request queue 中的请求， HBA 也一直运行在最大的 queue depth 深度。

# block层的trace events
在linux procfs的/sys/kernel/debug/tracing/events/block下，可以看到以下block相关的事件。
```shell
root/block/blk-core.c
├── block_getrq <- __get_request()#get a free request
├── block_sleeprq <- get_request()#get a free request
├── block_rq_requeue <- blk_requeue_request()
|# Description:
|#    Drivers often keep queueing requests until the hardware cannot accept
|#    more, when that condition happens we need to put the request back
|#    on the queue.
├── block_bio_frontmerge <- bio_attempt_front_merge()
├── block_bio_backmerge <- bio_attempt_back_merge()
├── block_plug <- blk_queue_bio()
├── block_unplug <- blk_flush_plug_list()#queue_unplug
├── block_bio_queue	<- generic_make_request_checks()
├── block_bio_remap <- blk_partition_remap()
├── block_rq_issue <- blk_peek_request()
└── block_rq_complete <- blk_update_request()

root/block/blk.c  
└── block_bio_complete <- bio_endio()

root/fs/buffer.c 
├── block_touch_buffer
└── block_dirty_buffer

root/block/bounce.c
└── block_bio_bounce <- blk_queue_bounce() 

root/block/elevator.c(blk-mq-sched.c)(blk-mq.c)
└── block_rq_insert <- __elv_add_request()

root/block/blk-merge.c
└── block_split <- blk_queue_split()

root/drivers/md/dm-rq.c
└── block_rq_remap <- map_request()
```
内核空间里面主要是给块层IO路径上的关键点添加tracepoint,然后借助于relayfs系统特性将收集到的数据写到buffer去。此时，我们来使能block的tracepoint
```shell
root@openmv:/sys/kernel/debug/tracing# echo > set_event
root@openmv:/sys/kernel/debug/tracing# echo 1 > events/block/enable 
root@openmv:/sys/kernel/debug/tracing# echo 1 > events/printk/enable 
root@openmv:/sys/kernel/debug/tracing# echo nomarkers > trace_options 
root@openmv:/sys/kernel/debug/tracing# echo 1 > tracing_on 
root@openmv:/sys/kernel/debug/tracing# echo 0 > trace      
root@openmv:/sys/kernel/debug/tracing# cat trace_pipe > /home/re.txt
```
打开```/home/re.txt```
```t
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              |  |      |   ||||       |         |
     jbd2/sda1-8-533   [000] .... 396695.376685: block_bio_remap: 8,0 FWFS 3905554656 + 8 <- (8,1) 3905552608
     jbd2/sda1-8-533   [000] .... 396695.376696: block_bio_queue: 8,0 FWFS 3905554656 + 8 [jbd2/sda1-8]
     jbd2/sda1-8-533   [000] .... 396695.376709: block_getrq: 8,0 FWFS 3905554656 + 8 [jbd2/sda1-8]
     jbd2/sda1-8-533   [000] d... 396695.376716: block_rq_insert: 8,0 FWFS 0 () 3905554656 + 8 [jbd2/sda1-8]
     ksoftirqd/3-23    [003] dns. 396695.377109: block_rq_issue: 8,0 WFS 0 () 3905554656 + 8 [ksoftirqd/3]
     ksoftirqd/3-23    [003] ..s. 396695.377480: block_rq_complete: 8,0 WFS () 3905554656 + 8 [0]
```

[0]: /img/perf/requestQ.png
[1]: /img/perf/requestLevel.jpg
[2]: /img/perf/blk_queue_bio.png
[3]: /img/perf/bio_merge.png
[4]: /img/perf/elv_merge.png
[5]: /img/perf/blk_flush_plug_list.png
[6]: /img/perf/unplug.png
