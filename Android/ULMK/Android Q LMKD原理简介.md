# Android Q LMKD原理分析
## 简介
lmk主要是为了在Android系统内存严重不足的时候，通过杀进程扩充系统内存，以保障用户关心的app能正常运行。  
lmk杀进程需要确保两点：  

1. 尽量少杀进程，后台尽量保留多个应用。这样在用户切换app时速度会较快，有较好的用户体验。
2. 内存也要尽量回收足够，以保证前台应用能流畅运行。   

所以lmk要在两点之间找到平衡。
## 主要框架与流程
![](https://github.com/reklaw-tech/reklaw_blog/blob/master/Android/ULMK/框架图.jpg)
lmkd存在于用户空间。  
lmkd需要从system server了解到最新的proc信息，proc开始或者结束，proc的oomadj是否发生变化。  
同时lmkd又需要从kernel获取当前内存压力变化，最新版本通过mem psi实现监听。也可以通过memcg的vmpressure实现监听。   

![](https://github.com/reklaw-tech/reklaw_blog/blob/master/Android/ULMK/整体流程图.jpg)  
lmkd的流程包括如下三步：

1. 获取配置信息，确定lmkd的一些算法
2. 初始化监听事件，包括监听system server和监听kernel内存压力
3. mainloop，开始消息处理运转工作  
## LMKD初始化
### 配置信息
| 属性名 | 含义 |
|:----|:---|
| ro.lmk.low | 内存压力为low时，开杀的最小adj |
| ro.lmk.medium | 内存压力为medium时，开杀的最小adj |
| ro.lmk.critical | 内存压力为critical时，开杀的最小adj |
| ro.lmk.debug | debug开关，打印杀进程信息 |
| ro.lmk.critical_upgrade | 是否允许升级内存压力，主要根据swap使用情况 |  
| ro.lmk.upgrade_pressure | 升级内存压力的判断阈值 |
| ro.lmk.downgrade_pressure | 降级内存压力的判断阈值 |  
| ro.lmk.kill_heaviest_task | 是否优先选择最重进程开杀 |
| ro.config.low_ram | 是否为低内存设备，Android Go设备 |
| ro.lmk.kill_timeout_ms | 距离上一次杀进程时间不足timeout，可能放弃本次杀操作 |
| ro.lmk.use_minfree_levels | 是否使用minfree_levels选择最小开杀adj |
| ro.config.per_app_memcg | 内存压力为low时，开杀的最小adj |
| ro.lmk.swap_free_low_percentage | swap使用情况判断阈值 |
| ro.lmk.enable_watermark_check | 是否检查水线 |
| ro.lmk.enable_preferred_apps | 内存压力为low时，开杀的最小adj |
| ro.lmk.use_psi | 是否使用psi作为内存压力监测接口 |

### 加载vendor lib
LMKD不仅仅支持从system prop中获取配置，还支持从vendor library中获取，library名为**perf_wait_get_prop**。  
如果存在该library，library获取到的配置会最终生效。
### 其他
通过mlockall将lmkd的内存页lock住。  
通过sched_setscheduler将lmkd进程设置为实时调度，具有更高优先级。
## 事件初始化
lmkd是通过epoll机制监听事件的，事件包括两个类型：1. socket事件 2. 内存压力事件。  
![](https://github.com/reklaw-tech/reklaw_blog/blob/master/Android/ULMK/事件处理.jpg)
event_handler_info分装了事件处理函数，设置进epoll event的data字段。event_handler_info根据作用不同，其data段的含义也不一样。  
sock_event_handler_info封装了socket的处理函数。  
### socket事件
lmkd中包括一个控制socket(ctrl_sock)负责listen，MAX_DATA_CONN个数据传输socket（data_sock）。  
控制socket的处理函数为ctrl_connect_handler，监听到客户端连接后从数据传输socket池中选取未使用的socket建立连接。并将数据传输socket的处理函数设置为ctrl_data_handle。  
ctrl_data_handle中根据传输过来的命令执行对应的操作。  
### memory事件
lmkd通过试探进入内核lmk模块路径（/sys/module/lowmemorykiller/parameters/minfree）的方式判断当前系统是否含义lmk模块。如果存在内核lmk模块，并且用户配置了enable_userspace_lmk为false，直接使用内核lmk。  
对于使用userspace lmk的情况，use_psi为true并且init_psi_monitor成功了，使用psi作为内存压力监测接口。  
否则调用init_mp_common，监听vmpressure事件。  
#### psi
| 压力等级 | 条件 |
|:-----|:---|
| LOW | 1000ms窗口期内70ms PSI_SOME |
| MEDIUM | 1000ms窗口期内100ms PSI_SOME |
| CRITICAL | 1000ms窗口期内70ms PSI_FULL |

(内存压力等级的定义不可更改)  
psi压力事件上报后回调mp_event_common处理函数。  
#### vmpressure
监听全局memcg的内存情况了解系统当前的压力情况。  
将"<evfd\> <mpfd\> <level\>"组成字符串写入cgroup.event_control实现内存压力监听。  
evfd通过eventfd系统生成，epoll通过监听该fd获取上报事件。  
mpfd是监控内存压力的事件类型，lmkd中监控memory.preesure_level。  
level作为压力事件类型的参数，lmkd中监控的为：low, medium和critical。  
##### vmpressure简介
vmpressure的计算其实是kernel做回收的时候。kernel首先计算pressure，计算公式为 **100 \* (1-reclaimed/scanned)**，再根据计算出来的pressure判断当前内存压力等级：  

1. pressure >= vmpressure_level_critical(95)  **内存处于critical等级**
2. pressure >= vmpressure_level_med(60)  **内存处于medium等级**
3. pressure < vmpressure_level_med  **内存处于low等级**

计算出内存等级后通过schedule_work挂在系统workqueue（keventd_wq）中执行通知userspace.
## 主循环
![](https://github.com/reklaw-tech/reklaw_blog/blob/master/Android/ULMK/mainloop.png)
### 与System Server交互
lmkd中维护了许多进程信息，用于后续选择杀进程，System Server会通过socket更新proc信息。  
lmkd中对进程的管理对象位proc结构体，通过一个hash表和ADJTOSLOT_COUNT个lru列表来维护所有的proc。  
hash表方便快速由pid找到对应的proc对象，lru列表则用来快速找到相同oomadj里存活时间最长的proc（**lmkd杀进程时会根据kill_heaviest_task判断，优先杀最大的还是优先杀最老的**）。    
![](https://github.com/reklaw-tech/reklaw_blog/blob/master/Android/ULMK/proc存储方式.jpg)
System Server通过socket传递给lmkd的cmd类型包括：  

| 命令 | 执行函数 | 作用 |
|:---|:-----|:---|
| LMK_PROCPRIO | cmd_procprio | 新建或更新Proc信息。如果开启per_app_memcg，根据oomadj设置其soft_limit_in_bytes |
| LMK_PROCREMOVE | cmd_procremove | 删除proc信息 |
| LMK_PROCPURGE | cmd_procpurge | 清空lmkd中所有proc信息 |
| LMK_GETKILLCNT | cmd_getkillcnt/ctrl_data_write | 向System Server返回杀进程次数 |
| LMK_TARGET | cmd_target | 根据socket传来数据，设置lowmem_minfree数组和lowmem_adj数组。lowmem_minfree用于后面选择最小开杀adj(min_score_adj) |

### 与kernel交互
内存压力事件上报后的处理事件为mp_event_common。  
事件处理主要分三步：

1. 确定内存压力等级 
2. 确定最低开杀的min_score_adj
3. 杀进程
#### 确定压力等级
**psi模式**只需要监听一个fd（/proc/pressure/memory），当前内存压力等级保留在epoll event中，并作为参数传递到了回调函数中，mp_event_common可以直接从参数data拿到当前level。   
**vmpressure模式**通过eventfd建立三个fd，分布监听三个内存压力等级。因此要遍历三个fd拿到当前最高的level，作为系统内存压力level。
#### 确定min_score_adj
lmkd首先通过meminfo_parse解析获得meminfo，zoneinfo_parse解析获得zoneinfo。  
根据配置min_score_adj的选择，根据是否use_minfree_levels有两种方式。
##### use_minfree_levels

```
// 刨除保留页，剩下页面个数
other_free = mi.field.nr_free_pages - zi.field.totalreserve_pages;
// memory info的nr_file_pages里包括了cached + swap_cached + buffers
// nr_file_pages为可以直接drop掉的page数量，后面为不可直接drop的page数量
if (mi.field.nr_file_pages > (mi.field.shmem + mi.field.unevictable + mi.field.swap_cached)) {
    other_file = (mi.field.nr_file_pages - mi.field.shmem -
                  mi.field.unevictable - mi.field.swap_cached);
} else {
    other_file = 0;
}

min_score_adj = OOM_SCORE_ADJ_MAX + 1;
for (i = 0; i < lowmem_targets_size; i++) {
    minfree = lowmem_minfree[i];
// 第一个条件，空闲内存很小，低于次级别的minfree
// 如果free memory很小，但是通过drop cache能得到很多内存，也没必要杀得很激进
if (other_free < minfree && other_file < minfree) {
    min_score_adj = lowmem_adj[i];
    // Adaptive LMK
    // 开启了Adaptive LMK，并且VMPRESS_LEVEL_CRITICAL代表内存压力大
    // i > lowmem_targets_size-4 代表内存现在很小
    // 此时加大回收力度，降低oomadj的阈值
    if (enable_adaptive_lmk && level == VMPRESS_LEVEL_CRITICAL &&
            i > lowmem_targets_size-4) {
        min_score_adj = lowmem_adj[i-1];
                }
        break;
    }
}
```
选择min_score_adj的值时，综合考虑当前free memory和file cache。  
free memory足够，或者虽然free memory不够，但是file cache drop掉也足够了，都没有必要激进杀进程。  
存在缺陷为，此处zoneinfo的totalreserve_pages未区分zone，可能导致free memory足够，但是分不到内存的情况。
该模式下不会调整memory警报级别，只是计算了min_score_adj。
##### 非use_minfree_levels
**调节内存压力等级**  

```
// 当前memory使用情况，不含swap
if ((mem_usage = get_memory_usage(&mem_usage_file_data)) < 0) {
    goto do_kill;
}
// 当前memory使用情况，含swap
if ((memsw_usage = get_memory_usage(&memsw_usage_file_data)) < 0) {
    goto do_kill;
}

// 这个指标类似于swapness，值越大，swap使用越少，剩余swap空间越大
mem_pressure = (mem_usage * 100) / memsw_usage;

if (enable_pressure_upgrade && level != VMPRESS_LEVEL_CRITICAL) {
    // 指标偏小说明swap使用很厉害，但仍然内存压力很大
    // 提高level，杀得更激进
    if (mem_pressure < upgrade_pressure) {
        level = upgrade_level(level);
    }
}
// swap_free_low_percentage为swap低阈值
// 此时swap空间还没到低阈值，有可操作空间
if (mi.field.total_swap && (mi.field.free_swap >=
    mi.field.total_swap * swap_free_low_percentage / 100)) {
    // 虽然有内存压力警报，但是swap还是足够的，不杀进程
    if (mem_pressure > downgrade_pressure) {
        return;
    } else if (level == VMPRESS_LEVEL_CRITICAL && mem_pressure > upgrade_pressure) {
	// swap空间足够的话，只有mem_pressure压力足够大，才会杀得更激进
        level = downgrade_level(level);
    }
}
```
swap剩余空间过小，level会升级，导致min oomadj更低，杀得更激进。  
swap剩余空间足够大，甚至可以放弃此次杀进程。  
对于critical级别的内存压力，只要swap足够大，也是有降级的空间。
该模式会调制memory警报级别。后续会根据警报级别调节min_score_adj。    

**最终确定min_score_adj**   
对于没有配置use_minfree_levels的情况，内存压力low时会调用record_low_pressure_levels，记录low等级时，历史最小空闲内存min_nr_free_pages和历史最大空闲内存max_nr_free_pages(为了保证最大free memory变化不大，只有当新的空闲内存在当前max_nr_free_pages的1.1倍内，才倍认可为新的max_nr_free_pages)。  
此时杀进程又分为检查水线路径和不检查水线路径。   

***对于不检查水线设置：*** 如果剩余内存比max_nr_free_pages还大，放弃杀进程。否则min_score_adj选取level_oomadj[level]，按该阈值oomadj杀进程。  
***对于检查水线设置：*** 根据水线检查函数zone_watermarks_ok确定min_score_adj，杀进程。  
***水线检查 zone_watermarks_ok***  

```
for (zone_id = 0; zone_id < MAX_NR_ZONES; zone_id++) {
  int margin;
  // 按zone类型解析zoneinfo，w存放zoneinfo信息
  nr = parse_one_zone_watermark(offset, &w);
  if (!nr)
    break;

  offset += nr;
  // zone并未管理任何page，跳过
  if (!w.present)
    continue;

  // 这边时从低级zone往高级zone累计所有的file page
  // 必要时drop cache得到便宜内存
  nr_file += w.inactive_file + w.active_file;

  // 当前zone在high watermark和cma刨掉后，还有多大空间，多大剩余内存？
  margin = w.free - w.cma - w.high;
  for (i = 0; i < MAX_NR_ZONES; i++)
    // lowmem_reserve[idx]是指当前zone面对来自idx zone申请需求时的保留内存空间
    // 如果当前zone剩余内存足够大，并且还能满足来自上层zone的借用需求，上层也安全了
    // 当然它只能保证最近一个能借用到的上层
    /* 
    举例： zone 1内存足够，保证了zone 2和zone 3都安全，
    但是zone 2不能保证zone 3安全，这就导致lowmem_reserve_ok[3]不ok了。
    但又如果zone 2如果一开始就不允许给3借用，那么lowmem_reserve_ok[3]还是ok的
    */
    if (w.lowmem_reserve[i] && (margin > w.lowmem_reserve[i]))
      lowmem_reserve_ok[i] = true;

  if (margin >= 0 || lowmem_reserve_ok[zone_id])
    continue;
  // 如果自身不能保证自己安全，底层也不能该zone安全
  // 根据到现在累计的page cache确定min oom adj
  return file_cache_to_adj(nr_file);
}
```
file_cache_to_adj函数就比较简单了，根据nr_file遍历比对lowmem_minfree，确定最后的min_score_adj。   
##### 总结
![](https://github.com/reklaw-tech/reklaw_blog/blob/master/Android/ULMK/oomadj确定.jpg)
#### 杀进程操作
**低内存手机**  
简单直接，直接根据警报级别，从level_oomadj确定阈值进行查杀。  
**非低内存手机**  
直接调用find_and_kill_process(min_score_adj)杀进程  
##### find_and_kill_process
![](https://github.com/reklaw-tech/reklaw_blog/blob/master/Android/ULMK/杀进程.jpg)

- 根据adj从大到小遍历lmkd中的proc列表，选择proc查杀
- 相同adj根据配置确定优先杀最重（rss最大）的还是优先杀最老(进lru最早)的proc
- 只要回收到内存就退出函数
- 如果没有回收到内存，有二次机会
- 二次机会之前遍历proc目录，更新lmkd中的proc列表信息  
**连续杀进程方法**  
虽然本次只杀了一个进程，但manloop中监听到内存压力事件就会调整epoll的超时时间。  
psi窗口1000ms，epoll的超时时间会调整到10ms，一次psi窗口可能有100次的超时触发。  
每次epoll的超时事件发生，会重新调用最近一次的内存压力处理事件继续处理，即mp_event_common，并且参数data还是最近一次的内存压力等级，以此实现继续杀内存操作。
## 内存指标
### zoneinfo
| 字段 | 含义 |
|:---|:---|
| nr_free_pages | 该zone空闲页数目 |
| nr_file_pages | 该zone文件页大小 |
| nr_shmem | 该zone中shmem/tmpfs占用内存大小 |
| nr_unevictable | 该zone不可回收页个数 |
| high | 该zone的高水位线 |
| protection | 该zone的保留内存 |

lmkd中zoneinfo_field_names保存了需要从zoneinfo中解析的字段，union zoneinfo则用来保存解析出来的数据。  
解析中使用了小技巧，zoneinfo为union，因此可以通过遍历zoneinfo_field_names的同时遍历zoneinfo的attr，实现快速解析。在使用时，又可以通过zone的field快速访问。  
zoneinfo中多计算了个**totalreserve_pages**，该值时根据high水线和protection保护页面数量（防止过度借出页面）共同计算得来(high水线 + protection选取最大保留页)。  
**lmkd中计算出来的zoneinfo为总大小，并未区分各个zone**
### meminfo
| 字段 | 含义 |
|:---|:---|
| MemFree | 系统尚未使用的内存 |
| Cached | 文件页缓存，其中包括tmpfs中文件（未发生swap-out） |
| SwapCached | 匿名页或者shmem/tmpfs,被swapout过，当前swapin后未改变,如果改变会从SwapCached删除 |
| Buffers | io设备占用的缓存页，也统计在file lru |
| Mapped | 正在与用户进程关联的文件页 |

shmem比较特殊，基于文件系统所以不算匿名页，但又不能pageout，因此在内存中被统计进了Cached （pagecache）和Mapped（shmem被attached），但lru里是放在anon lru，因为可能会被swap out。  
lmkd的meminfo中也多计算了一个字段**nr_file_pages**，该值包括cached + swap_cached + buffers。可以理解为能够被drop的文件页。  
### memcg
| 字段 | 含义 |
|:---|:---|
| memory.usage_in_bytes | 该memcg的内存（不含swap）使用情况 |
| memory.memsw.usage_in_bytes | 该memcg的内存（含swap）使用情况 |
### 进程信息
进程rss信息获取："/proc/***pid***/statm"  
统计的数据依次为：虚拟地址空间大小，rss，共享页数，代码段大小，库文件大小，数据段大小，和脏页大小（单位为page）。  
进程状态信息： "/proc/***pid***/status"  
进程统计信息： "/proc/***pid***/stat"   
lmkd比较关心第10位pgfault，12位pgmajfault，22位进程开始时间，rss大小（单位page）。  

