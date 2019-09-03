# 常用命令

linux常用
```
sed -i '1,3d' test11.txt
awk -F "|" '{print $2"\t"$3"\t"$4"\t"$5}'
lsb_release -a // 查看系统

// 查看内存
free -g
cat /proc/meminfo |grep MemTotal
cat /proc/meminfo |grep MemFree

ln -s 系统源地址  软链接目标地址
sudo chown -R magneto:magneto tmp
split -b 100k test.tar.gz test_split/split_
cat split_* >test.tar.gz

tar命令
　　解包：tar zxvf FileName.tar
　　打包：tar czvf FileName.tar DirName
zip -r mydata.zip mydat
unzip -n nerdtree.zip -d nerdtree

for i in *;do mv $i /opt/media/img_20171020;done
```

IO

```
iostat -x 1
dd bs=1M count=500 if=/dev/zero of=test.dd conv=sync // 磁盘测试
```

查看进程端口

```
netstat -nap | grep 42877
```

/dev/null: 空设备文件

```
> ?：代表重定向到哪里，例如：echo "123" > /home/123.txt
1 ?：表示stdout标准输出，系统默认值是1，所以">/dev/null"等同于"1>/dev/null"
2 ?：表示stderr标准错误
& ?：表示等同于的意思，2>&1，表示2的输出重定向等同于1

1 > /dev/null 2>&1?
// 语句含义：
1 > /dev/null：首先表示标准输出重定向到空设备文件，也就是不输出任何信息到终端，说白了就是不显示任何信息。
2>&1 ：接着，标准错误输出重定向（等同于）标准输出，因为之前标准输出已经重定向到了空设备文件，所以标准错误输出也重定向到空设备文件。
```

常用Hadoop命令

```
hadoop dfs -getmerge /test_output /home/magneto/tw3.1/hehe.txt // 把HDFS上的多个文件 合并成一个本地文件
hadoop dfs -ls /elasticsearch
hadoop dfs -du -h /elasticsearch
hadoop dfs -get /elasticsearch/voxer_snapshot /data07/snapshot/es
hadoop dfs -put /opt/voxer_snapshot /elasticsearch
hadoop dfs -rm -r /elasticsearch/voxer_snapshot
hadoop dfs -mkdir -p /es

web-ui:http://ip:8088/
yarn:http://ip:50070
```

Gradle命令

```
./gradlew clean build -xtest
./gradlew build --refresh-dependencies
```

Git
```
// Git地址重定向
git remote -v
git remote set-url origin git@192.168.5.19:deepclue/dataarch.git

查看各分支状态
git remote show origin
清理无用分支
git remote prune origin
```

磁盘挂载

```
parted /dev/sdc
mklabel gpt 按q退出
fdisk /dev/sdc 输入n 然后一直回车默认 最后输入w保存
mkfs.xfs -f /dev/sdc
mount -t xfs /dev/sdc /data01
/etc/fstab中按照格式加入磁盘配置 /dev/sdc /data02 xfs defautls 0 0
```

Java环境变量配置

```
yum install -y /opt/ext/jdk-*-linux-x64.rpm
alternatives --install /usr/bin/java java /usr/java/jdk1.8.0_72/bin/java 1
alternatives --config java
```

jar包启动

```
java -jar X.jar --logging.level.root=debug // 修改日志打印级别
nohup java -jar X.jar --spring.cloud.config.profile=debug --spring.profiles.active=debug &
```

ssh key

```
ssh-keygen -t rsa
ssh-copy-id -i ~/.ssh id_rsa.pub targetIp 
```

一个简单的shell程序

```
// 读文件的行并搜索目录拷贝相应文件
cat ../Untitled@dbo.txt | while read line
do
flag=`ls -rt $line*`
echo $flag
`cp $flag ../ll`
echo `end`
done
```