第一次条件

ud 线程 500 ，MALLOC_ARENA_MAX=8 反应很慢

第二次条件

ud线程500，MALLOC_ARENA_MAX=32 

重启时间0:10 ，200个进程运行folder-detail，运行结束时间1:30，working维持在300左右，但是中间有无法连接的情况，连接数偏高，内存使用没有查看

第三次条件

ud线程1000，MALLOC_ARENA_MAX=32

重启时间2:10，200个进程运行folder-detail，结束时间3:05，working维持在250左右，初始阶段稍高，连接数维持在2500左右，内存占用

第四次条件

ud线程1000，MALLOC_ARENA_MAX=16

重启时间3:07，200个进程运行folder-detail ，结束4:00，working比较高，最高去到1000+，稳定在400左右，连接数位置在2500左右，内存33g

第五次条件

ud线程1000，MALLOC_ARENA_MAX=32

重启时间4:16，