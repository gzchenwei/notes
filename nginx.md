###　nginx 信号集

２种方式可以向ｍａｓｔｅｒ发送信号，

*  nginx -s signal 
原理：产生一个新进程，该进程通过nginx.pid获得master pid，然后把信号发送到master,之后退出
*  kill 
operation|signal
----|----
reload|sighup
reopen|sigusr1
stop|sigterm
quit|sigquit
hot update|sigusr2 ＆ sigwinch＆ sigquit

    * stop vs quit
       stop 发送sigterm信号，强制退出
       quit发送sigquit，优雅退出
       worker进程在收到sigquit信息后，会关闭监听的套接字，关闭当前空闲的连接，然后提前处理所有的定时器事件，最后退出，没有特殊情况都应该使用quit，而不是stop
    * reload
       master收到sighup，会重新进行配置文件解析，共享内存申请等一系列工作，然后产生一批新的worker进程，最后向旧的worker发送sigquit
    * reopen
       master进程收到sigusr1，会重新打开所有已经打开的文件(比如日志），然后向每个worker发送sigusr1，worker收到信号后，会执行同样的操作，主要用于日志切割。
    * hot update
       某些情况我们需要进行二进制更新，流程如下：
           + 给当前的master发送sigusr2，master会重命名nginx.pid为nginx.pid.oldbin
           + master fork新的进程，新进程通过execve系统调用，使用新的nginx elf文件替换当前的进程映像，成为新的master
           + 新的master起来后，会进行配置文件解析等操作，然后fork出新的worker进程开始工作
           + 接着我们向旧的master发送sigwinch信号，然后旧的master会向它的worker发送sigquit信息，使旧的worker进程退出（发送winch和quit都会使worker进程退出，但是winch不会导致master退出）
           + 如果我们认为旧的master完成使用，可以向它发送sigquit信号
  
work进程如何接受master进程的信号消息
master通过管道实现的nginx channel
       
```
 mv access.log access.log.0
 kill -USR1 `cat master.nginx.pid`
 sleep 1
 gzip access.log.0
```
  这里sleep 1 是必须的，因为master向worker发送sigusr1消息到worker进程真正重新打开access.log之间，有一段时间窗口，此时worker进程还是向旧文件写入
    
   
       
     
 
