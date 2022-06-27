# Linux Swap 如何手工干预

## 1.添加swap分区

```sh
dd if=/dev/zero of=/data/swapfile bs=1024 count=4096k
```


    if(即输入文件,input file)，dev/zero 是Linux的一种特殊字符输入设备，用来创建一个指定长度用于初始化的空文件。
    of(即输出文件,output file)。 /data/swapfile 是 swap 文件地址。
    bs=1024 ：单位数据块同时读写块字节大小为1024个字节即。
    count=4096K ：数据块数量为4096*1024。
    计算出swap分区的容量为：1KB*4096*1024=4G。


转换为swap分区：

```sh
mkswap /data/swapfile
```

挂载并激活分区：

```sh
swapon /data/swapfile
```

设置权限为root可操作

```sh
chmod -R 0600 /data/swapfile
```

设置开机自动挂载该分区：

```sh
vi /etc/fstab 
UUDI=swapfile的UUID swap swap defaults 0 0
```

## 2.删除某swap分区

先停止正在使用swap分区：

```sh
swapoff /data/swapfile
```

删除swap分区文件

```sh
rm -rf /data/swapfile
```
删除 /etc/fstab 中的配置
UUDI=swapfile的UUID swap swap defaults 0 0

## 3.更改Swap配置，swappiness值越高系统对swap分区的使用优先级越高，默认为30.

查看当前的swappiness数值：

```sh
cat /proc/sys/vm/swappiness
```

修改swappiness值，这里以10为例。
sysctl vm.swappiness=10
永久生效
echo "vm.swappiness = 10" >> /etc/sysctl.conf