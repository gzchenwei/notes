mysql 出现问题时,需要运行如下命令,采集数据

```
echo "show full processlist\G"|/home/coremail/mysql/bin/mysql -ucoremail -pmailc0re -h10.13.0.3 -P3306
echo "show engine innodb state\G"|/home/coremail/mysql/bin/mysql -ucoremail -pmailc0re -h10.13.0.3 -P3306
iostat -x -k -d 2 /dev/mapper/ms1_00p1
```

可以执行下strace跟踪下系统调用

```
strace -c -p mysqlpid
```

