bind 9.11.2

#### 步骤

* 官网下载安装包
* 编译安装

```
yum install openssl-devel
yum install make gcc
./configure --prefix=/var/named --sysconfdir=/etc/named --disable-ipv6 --enable-threads
make && make install 
```

* 拷贝目录到chroot

```
cd /var/named/chroot
cp -a ../{bin,sbin,dev,share,include,etc,var} .
cd /var/named/chrome/var/named
cp -a /usr/share/doc/bind-9.9.4/sample/var/named/* .
mkdir dynamic
chmod 777 dynamic data
cd dynamic
touch managed-keys.bind
```

* 创建随机文件

```
cd /var/named/chroot/dev
mknod /var/named/chroot/dev/random c 1 8
chmod 644 /var/named/chroot/dev/randomc
```

